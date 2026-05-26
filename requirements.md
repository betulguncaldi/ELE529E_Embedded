# Requirements Specification for Obstacle Detection ADAS System

## 1. Functional Requirements (FR)

| ID   | Description                                                                                                                     | Priority |
|----- |---------------------------------------------------------------------------------------------------------------------------------|----------|
| FR1  | The system shall detect obstacles in the path of the vehicle.                                                                   | High     |
| FR2  | The system shall provide real-time distance measurements.                                                                       | High     |
| FR3  | The system shall alert the driver when an obstacle is detected within a certain range.                                          | High     |
| FR4  | The system shall activate a green LED indicator when the detected obstacle distance is greater than 65 mm.                      | High     |
| FR5  | The system shall activate a red LED indicator when the detected obstacle distance is less than or equal to 65 mm.               | High     |
| FR6  | The system shall measure the obstacle distance with an accuracy tolerance of 5 mm within the operational range.                 | High     |
| FR7  | The I2C communication interface shall be configured at a clock frequency of 400 kHz (Fast Mode).                                | High     |
| FR8  | The I2C communication bus shall implement 10 kohm external pull-up resistors on the SDA and SCL lines.                          | Medium   |
| FR9  | The shutdown pin of the VL6180X sensor shall have an external 10 kohm pull-up resistor.                                         | Medium   |
| FR10 | The I2C communication bus shall be protected using a FreeRTOS Mutex mechanism to prevent shared resource conflicts and data     | High     |
corruption.
| FR11 | Moving Average Filter shall be implemented in software to suppress sensor noise and eliminate sudden measurement spikes.        | High     |
| FR12 | The system shall transmit the processed distance data and system status messages to an external PC via the UART interface at    | Medium   |
a baud rate of 115200 bps.

## 2. Non-Functional Requirements (NFR)

| ID   | Description                                                                                                                     | Priority |
|----- |---------------------------------------------------------------------------------------------------------------------------------|----------|
| NFR1 | The system shall be compliant with safety standards for automotive applications.                                                | High     |
| NFR2 | The system shall be maintainable and upgradable.                                                                                | Medium   |
| NFR3 | The measurement task shall sample the sensor at a cycle time of 20 ms to ensure low-latency obstacle detection.                 | High     |

## 3. General Description

The Obstacle Detection ADAS (Advanced Driver-Assistance System) is designed to enhance vehicle safety by detecting obstacles and providing timely alerts to the driver. Utilizing the STM32F4C11 microcontroller and the VL6180X distance sensor, the system operates with FreeRTOS to ensure efficient multitasking and real-time performance.
