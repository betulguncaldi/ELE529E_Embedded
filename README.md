# Host unit tests (`test/unit`)

This directory contains **host-side unit tests** for the custom device drivers you wrote under `Drivers/dev_*` (soft PWM, LED, button, serial). Tests run on your development machine with [Google Test](https://github.com/google/googletest) and compile the **same driver source files** used on the STM32 target, but link against **fakes** instead of the real STM32 HAL/CMSIS tree.

They are deliberately **not** on-board tests, FreeRTOS task tests, or full-application integration tests.

---

## Directory layout

```
test/unit/
├── CMakeLists.txt          # Build: GTest fetch, driver executables, CTest, coverage target
├── run_unit_tests.sh       # One-shot: configure, build, test, coverage, metrics summary
├── scripts/
│   ├── run_coverage.sh     # gcovr HTML + JSON/text (Drivers/ only)
│   └── generate_metrics.py # Merges GTest XML + coverage → metrics_summary.{json,md}
├── README.md               # This file
├── scenarios/
│   └── scenarios.yaml      # Scenario catalog (IDs, given/when/then, test level)
├── tests/                  # GTest sources (one executable per driver)
│   ├── softpwm_driver_test.cpp
│   ├── led_driver_test.cpp
│   ├── button_driver_test.cpp
│   └── serial_driver_test.cpp
├── support/
│   ├── include/            # Minimal STM32/HAL headers + board.h for host
│   ├── fakes/              # HAL_GPIO_*, HAL_TIM_*, HAL_UART_* implementations + metrics
│   └── stubs/              # irq context tables, SystemCoreClock
└── metrics/                # Placeholder; real output goes to build/unit/metrics/
```

**Production code under test** (unchanged paths, compiled into each test binary):

| Driver   | Source |
|----------|--------|
| Soft PWM | `Drivers/dev_softpwm/src/softpwm_driver.c` |
| LED      | `Drivers/dev_led/src/led_driver.c` |
| Button   | `Drivers/dev_button/src/button_driver.c` |
| Serial   | `Drivers/dev_serial/src/serial_driver.c` |

**Out of scope here** (other test levels): `Core/Src/main.c` (FreeRTOS tasks, crypto), `*_TEST()` demo functions, `stm32f1xx_it.c` ISR wiring, Cube HAL/CMSIS, Middlewares.

---

## Build system

### CMake graph

1. **Root** `CMakeLists.txt` — enables testing and adds `test/unit` as a subdirectory.
2. **`test/unit/CMakeLists.txt`** — defines the unit-test world:
   - **FetchContent** downloads Google Test 1.14.0 (and Google Mock).
   - **`unit_test_support`** static library: HAL fakes + stubs.
   - **`add_driver_unit_test()`** — one executable per driver: `tests/<name>.cpp` + one `Drivers/.../*.c`.
   - **CTest** registers each executable; GTest writes JUnit-style XML to `build/unit/metrics/`.
   - **`unit_test_coverage`** — gcovr HTML + JSON/text (Clang/macOS friendly).
   - **`unit_test_metrics`** — `metrics_summary.json` / `.md` (tests + coverage).
   - **`unit_test_report`** — both targets above.

### Include path strategy

Host builds use `test/unit/support/include/` **before** the real board/HAL tree:

- `board.h` → thin wrapper around `board_unit_test.h` (same pin/timer macros, no busy-wait `delayMs`).
- Minimal `stm32f103xb.h`, `stm32f1xx_hal_*.h` — only types and symbols the drivers need.
- `hal_fakes.c` implements `HAL_GPIO_*`, `HAL_TIM_*`, `HAL_UART_*` and records call counts for assertions.

Driver `.c` files are **not** copied; they are compiled directly from `Drivers/` so refactors stay in sync with firmware.

### Options

| CMake option | Default | Effect |
|--------------|---------|--------|
| `UNIT_TEST_COVERAGE` | `ON` | Adds `--coverage` / `-O0` for gcov/gcovr reports |

### Build output locations

| Artifact | Path |
|----------|------|
| Build tree | `build/unit/` |
| Test binaries | `build/unit/test/unit/<name>` (exact path may vary by generator) |
| GTest XML | `build/unit/metrics/*_results.xml` |
| Coverage HTML | `build/unit/metrics/coverage/html/index.html` |
| Coverage JSON/text | `build/unit/metrics/coverage/coverage_summary.{json,txt}` |
| Combined metrics | `build/unit/metrics/metrics_summary.{json,md}` |

---

## Running tests

### Quick run

```bash
./test/unit/run_unit_tests.sh
# or: bash test/unit/run_unit_tests.sh
```

Use **bash** or execute the script directly (`./…`). Do not use `sh run_unit_tests.sh` on macOS: `/bin/sh` is not bash (the script re-execs itself with bash if you do, but `./` or `bash` is clearer).

### Manual steps

```bash
cmake -S . -B build/unit -DUNIT_TEST_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
cmake --build build/unit
ctest --test-dir build/unit --output-on-failure
```

### Coverage and code metrics

`run_unit_tests.sh` installs **gcovr** via Homebrew if needed, then produces:

| Output | Contents |
|--------|----------|
| `coverage/html/index.html` | Line-by-line HTML (per driver file) |
| `coverage/coverage_summary.json` | Line/function/branch % per file (gcovr) |
| `coverage/coverage_summary.txt` | Human-readable table |
| `metrics_summary.json` | Tests (pass/fail, timing) + coverage totals |
| `metrics_summary.md` | Same data as Markdown for CI/docs |

Manual:

```bash
brew install gcovr   # once
cmake --build build/unit --target unit_test_report
open build/unit/metrics/coverage/html/index.html
open build/unit/metrics/metrics_summary.md
```

Coverage is scoped to **`Drivers/`** only (your driver code), excluding tests, fakes, and googletest.

### Run a single suite

```bash
./build/unit/test/unit/softpwm_driver_test
./build/unit/test/unit/softpwm_driver_test --gtest_filter='*SetDuty*'
```

---

## Scenarios and test style

Each test case is documented in three layers:

1. **Comment block** above the test — scenario ID (e.g. `SOFTPWM-004`), short name, and **rationale** (why this case matters).
2. **Given / When / Then** sections inside the test body.
3. **`scenarios/scenarios.yaml`** — machine-readable catalog for traceability (CI, reviews, safety cases).

Example pattern:

```cpp
// Scenario LED-003 (unit): SET_LED drives GPIO high through HAL fake.
// Rationale: ioctl must map command to HAL write without relying on hardware.
TEST_F(LedDriverTest, Ioctl_SetLed_WritesPinSet_GivenValidLedIndex) {
  // Given ...
  // When ...
  // Then ...
  EXPECT_EQ(hal_fakes_gpio_metrics()->last_pin_state, GPIO_PIN_SET);
}
```

Scenario IDs in YAML match the comments in `tests/*.cpp`.

---

## Important design choices

| Topic | Choice | Reason |
|-------|--------|--------|
| **Where tests run** | Host (macOS/Linux) | Fast feedback, no debugger/flasher required |
| **HAL** | Fakes with metrics | Isolate driver logic; assert *that* HAL was called correctly |
| **Board config** | `board_unit_test.h` | Same `*_TOTAL_NUM` / pin macros as `Board/board.h` |
| **IRQ context** | `irq_context_stub.c` | Satisfies `extern gpio_irq_context` / `serial_irq_context` without `stm32f1xx_it.c` |
| **Google Mock** | Linked, light use | Available for future interface mocks; fakes cover current HAL surface |
| **Per-driver binary** | 4 executables | Failures are localized; link only the driver under test |

---

## Why these tests fit the unit-test level

In the V-model, **unit tests** verify a single module in isolation, with dependencies replaced by test doubles, in a deterministic environment. This suite matches that definition in practice:

### 1. Single unit under test (SUT)

Each executable links **one** driver implementation (`softpwm_driver.c`, etc.) plus test code and fakes. Assertions target that driver’s public API (`SoftPwm_Open`, `Led_Ioctl`, `BUTTON_Read`, …), not `main()` or task loops.

### 2. Isolation from hardware and platform

- No STM32 chip, no GPIO toggling on real pins, no UART cable.
- No CMSIS, no Cube `Drivers/STM32F1xx_HAL_Driver`, no startup code.
- `SystemCoreClock` and NVIC calls are stubs; timer/UART behavior is simulated.

If a test fails, you debug **driver logic or ioctl contracts**, not wiring or clock tree.

### 3. Controlled dependencies (test doubles)

| Real dependency | In unit tests |
|-----------------|---------------|
| `HAL_GPIO_Init/Write/Read` | `hal_fakes.c` — records calls and programmable pin state |
| `HAL_TIM_Base_*` | Fake — counts start/stop/init |
| `HAL_UART_Transmit/Receive` | Fake — injectable `HAL_OK` / `HAL_ERROR` |
| EXTI / IRQ vectors | Not executed; handler **registration** is tested via context structs only |

That is classic **fake/stub** isolation, not “call the real HAL on the board.”

### 4. Deterministic and fast

- No `delayMs()` busy waits (`board_unit_test.h` makes them no-ops).
- No FreeRTOS scheduler, semaphores, or `vTaskDelay`.
- Full suite runs in **under a few seconds** on the host.

Unit tests should be repeatable and cheap to run on every commit.

### 5. Small, focused examples

Each test checks **one behavior** (e.g. invalid duty → `E_SOFTPWM_ERR_INVALID_DUTY`, or register IRQ handler → context slot filled). That maps to one scenario row in `scenarios.yaml`, not an end-to-end “press button, LED blinks, UART prints” flow.

### 6. What we explicitly do *not* claim as unit tests

These belong to **integration** or **system** test levels and are excluded on purpose:

| Item | Why not unit |
|------|----------------|
| `LED_TEST`, `BUTTON_TEST`, `SOFTPWM_TEST`, `SERIAL_TEST` | Multi-step demos with real delays and hardware |
| `main.c` / FreeRTOS tasks | Multiple modules + scheduler |
| `Encrypt_AES_CBC` / `cmox_*` | Crypto middleware + platform init |
| `HAL_TIM_PeriodElapsedCallback` from real ISR | Needs vector table and hardware timer (we only call the function directly to test the **callback body** in isolation) |

Calling those “unit tests” would blur the test pyramid; keeping them separate preserves clear failure diagnosis.

### 7. Checklist (unit-test characteristics)

| Characteristic | This suite |
|----------------|------------|
| Tests one module | Yes — per-driver executable |
| Dependencies substituted | Yes — HAL fakes/stubs |
| No external I/O | Yes — host process only |
| Fast | Yes — sub-second per binary |
| Deterministic | Yes — fake state reset in `SetUp()` |
| White-box on your code | Yes — covers ioctl branches, error codes |
| Traceable scenarios | Yes — IDs in code + YAML |

---

## Adding a new unit test

1. Add a `TEST_F` in the appropriate `tests/*_driver_test.cpp` (or create a new file and register it in `CMakeLists.txt` via `add_driver_unit_test`).
2. Add a matching entry to `scenarios/scenarios.yaml`.
3. If the driver calls new HAL APIs, extend `support/fakes/hal_fakes.c` and the minimal headers.
4. Rebuild and run `ctest --test-dir build/unit`.

---

## Prerequisites

- **CMake** ≥ 3.16  
- **C/C++ compiler** (Clang or GCC) with C++14 support  
- **Network** on first configure (Google Test download via FetchContent)  
- **Coverage:** `gcovr` (`brew install gcovr`; auto-installed by `run_unit_tests.sh` when Homebrew is available)  

STM32CubeIDE / arm-none-eabi toolchain is **not** required to build or run these tests.

---

## Related firmware build

On-target firmware is still built with **STM32CubeIDE** (or your existing Makefile flow under `Debug/`). The host unit-test build is **orthogonal**: it does not replace flashing or HIL tests; it complements them by catching logic errors earlier on the driver layer you own.

## Secure serial application

Firmware application code lives in `Core/App/` (protocol, crypto, comm, LED app). PC integration client: [`tools/pc_client/README.md`](../../tools/pc_client/README.md). System E2E UI: [`test/system/README.md`](../system/README.md).
