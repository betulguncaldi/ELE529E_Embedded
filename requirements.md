# Requirements Specification for Obstacle Detection ADAS System

## 1. Functional Requirements (FR)

| ID  | Description                                                   | Priority |
|-----|---------------------------------------------------------------|----------|
| FR1 | The system shall detect obstacles in the path of the vehicle. | High     |
| FR2 | The system shall provide real-time distance measurements.     | High     |
| FR3 | The system shall alert the driver when an obstacle is detected within a certain range. | High     |
| FR4 | The system shall support multiple distance sensors.           | Medium   |
| FR5 | The system shall integrate with the vehicle's braking system for emergency stops. | High     |

## 2. Non-Functional Requirements (NFR)

| ID  | Description                                                     | Priority |
|-----|---------------------------------------------------------------|----------|
| NFR1| The system shall operate in real-time with a maximum latency of 100ms.  | High     |
| NFR2| The system shall consume less than 100mA of current.         | Medium   |
| NFR3| The system shall be compliant with safety standards for automotive applications. | High     |
| NFR4| The system shall be maintainable and upgradable.              | Medium   |
| NFR5| The system shall function in a temperature range of -20°C to +70°C. | High     |

## 3. General Description

The Obstacle Detection ADAS (Advanced Driver-Assistance System) is designed to enhance vehicle safety by detecting obstacles and providing timely alerts to the driver. Utilizing the STM32F4C11 microcontroller and the VL6180X distance sensor, the system operates with FreeRTOS to ensure efficient multitasking and real-time performance.
