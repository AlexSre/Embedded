# Dual-STM32 UART Master-Slave Motor Control Prototype

## Overview

This project is a small embedded systems prototype built with two **STM32 Nucleo-F446RE** development boards. The goal is to create a simple master-slave architecture that can later be extended into an autonomous robot control platform for line following, motor control, telemetry and real-time embedded communication.

The current version implements a basic UART communication test:

- The **Master** board periodically sends a `PING` message.
- The **Slave** board waits for `PING`.
- When the Slave receives the message, it toggles the onboard LED and replies with `PONG`.
- The Master waits for `PONG` and toggles its onboard LED when the response is valid.

This project is designed as a first step toward a more advanced NXP Cup-style autonomous robotics system.

---

## Hardware Used

- 2 × STM32 Nucleo-F446RE boards
- 1 × mini-USB cable
- Jumper wires
- Optional: breadboard, external LEDs, motor driver, DC motors, sensors

---

## Boards Used

### STM32 Nucleo-F446RE

Main specifications:

- MCU: STM32F446RET6
- Core: Arm Cortex-M4
- Clock frequency: up to 180 MHz
- Flash: 512 KB
- SRAM: 128 KB
- Debug/programming: onboard ST-LINK
- Communication peripherals: USART, SPI, I2C, CAN, USB
- Timers and PWM support
- Onboard user LED: LD2 on PA5

---

## System Architecture

```text
+-------------------+        UART         +-------------------+
|   STM32 Master    |  TX ----------> RX  |    STM32 Slave    |
|                   |  RX <---------- TX  |                   |
| Sends PING        |                     | Receives PING     |
| Waits for PONG    |                     | Sends PONG        |
| Toggles LD2       |                     | Toggles LD2       |
+-------------------+                     +-------------------+
```

Power setup used during testing:

```text
Laptop USB -> Master board
Master 5V  -> Slave 5V
Master GND -> Slave GND
```

UART wiring:

```text
Master PA2 / USART2_TX / D1 -> Slave PA3 / USART2_RX / D0
Master PA3 / USART2_RX / D0 -> Slave PA2 / USART2_TX / D1
Master GND                  -> Slave GND
```

---

## Software Tools

- STM32CubeMX
- STM32CubeIDE
- STM32CubeProgrammer
- STM32CubeF4 firmware package
- ST-LINK USB driver

---

## Peripheral Configuration

Both STM32 boards use the same basic configuration.

### GPIO

| Function | Pin | Configuration |
|---|---|---|
| Onboard LED LD2 | PA5 | GPIO Output |

### USART2

| Signal | Pin | Arduino Header |
|---|---|---|
| USART2_TX | PA2 | D1 |
| USART2_RX | PA3 | D0 |

USART2 settings:

```text
Baud rate: 115200
Word length: 8 bits
Parity: None
Stop bits: 1
Mode: TX/RX
```

---

## Current Firmware Behavior

### Master Firmware

The Master board:

1. Sends `PING` over USART2 every second.
2. Waits up to 1 second for the response `PONG\r\n`.
3. If the response is valid, toggles the onboard LED on PA5.

### Slave Firmware

The Slave board:

1. Waits for 4 bytes over USART2.
2. Checks whether the received message is `PING`.
3. If valid, toggles the onboard LED on PA5.
4. Sends back `PONG\r\n`.

---

## Example Master Code

```c
uint8_t msg[] = "PING";
uint8_t rx_buffer[6];
uint8_t expected[] = "PONG\r\n";

while (1)
{
    HAL_UART_Transmit(&huart2, msg, sizeof(msg) - 1, HAL_MAX_DELAY);

    if (HAL_UART_Receive(&huart2, rx_buffer, 6, 1000) == HAL_OK)
    {
        if (rx_buffer[0] == expected[0] &&
            rx_buffer[1] == expected[1] &&
            rx_buffer[2] == expected[2] &&
            rx_buffer[3] == expected[3] &&
            rx_buffer[4] == expected[4] &&
            rx_buffer[5] == expected[5])
        {
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        }
    }

    HAL_Delay(1000);
}
```

---

## Example Slave Code

```c
uint8_t rx_buffer[4];
uint8_t response[] = "PONG\r\n";

while (1)
{
    HAL_UART_Receive(&huart2, rx_buffer, 4, HAL_MAX_DELAY);

    if (rx_buffer[0] == 'P' &&
        rx_buffer[1] == 'I' &&
        rx_buffer[2] == 'N' &&
        rx_buffer[3] == 'G')
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        HAL_UART_Transmit(&huart2, response, sizeof(response) - 1, HAL_MAX_DELAY);
    }
}
```

---

## How to Build and Flash

1. Open each project in STM32CubeIDE.
2. Build the project.
3. Connect one board at a time using the mini-USB cable.
4. Flash the Slave firmware to the Slave board.
5. Disconnect the Slave board.
6. Flash the Master firmware to the Master board.
7. Keep the Master connected by USB.
8. Power the Slave from the Master using 5V and GND.
9. Connect UART TX/RX lines between the boards.

A successful flash should show a message similar to:

```text
Download verified successfully
```

---

## Test Procedure

1. Program the Slave board.
2. Program the Master board.
3. Connect the two boards:

```text
Master 5V  -> Slave 5V
Master GND -> Slave GND
Master D1  -> Slave D0
Master D0  -> Slave D1
```

4. Power the Master through USB.
5. Observe the onboard LEDs.

Expected behavior:

- The Slave LED toggles when it receives `PING`.
- The Master LED toggles when it receives `PONG`.

---

## Troubleshooting

### Only LD1 flickers red

LD1 is usually related to ST-LINK communication. The important user LED is LD2 on PA5.

If LD2 does not toggle, check:

- TX is connected to RX, not TX to TX.
- RX is connected to TX.
- GND is common between both boards.
- Both boards are configured with USART2 at 115200 baud.
- The Slave is powered correctly.
- The right firmware was flashed to the right board.

### No ST-LINK detected

Check:

- USB cable is connected to the board being programmed.
- Cable supports data, not only charging.
- STM32CubeProgrammer is closed while CubeIDE is trying to flash.
- Board appears in Device Manager.

### Slave does nothing

This can be normal if the Slave is waiting inside:

```c
HAL_UART_Receive(&huart2, rx_buffer, 4, HAL_MAX_DELAY);
```

It will not continue until it receives exactly 4 bytes.

---

## Planned Extensions

This project is intended to evolve into a more advanced autonomous robot control system.

Planned improvements:

- Replace `PING/PONG` with a binary packet protocol.
- Add start byte, command ID, payload and CRC.
- Implement ACK/NACK communication.
- Add PWM motor commands.
- Use the Slave as a motor controller.
- Use the Master as a decision-making unit.
- Add PID line-following logic.
- Add UART telemetry for debugging.
- Add sensor calibration for line sensors.
- Add external motor driver and DC motors.

Future packet format:

```c
typedef struct
{
    uint8_t start;      // 0xA5
    uint8_t command;    // 0x01 = SET_PWM
    uint8_t left_pwm;   // 0-100
    uint8_t right_pwm;  // 0-100
    uint8_t crc;        // XOR checksum
} motor_packet_t;
```

---

## Relevance to NXP Cup Bootcamp

This project is relevant to autonomous robotics because it demonstrates:

- Embedded C programming
- UART communication between microcontrollers
- Master-slave architecture
- Basic distributed control
- GPIO control
- Debugging with ST-LINK
- Preparation for PWM-based motor control
- Preparation for PID-based line following

The final goal is to extend this into a modular autonomous robot platform where one microcontroller handles high-level decision-making and another handles low-level motor control.

---

## What I Learned

Through this project, I practiced:

- Creating STM32 projects using STM32CubeMX and STM32CubeIDE
- Configuring GPIO and USART2
- Programming and flashing STM32 Nucleo boards with ST-LINK
- Connecting two microcontrollers through UART
- Understanding TX/RX crossing and common ground requirements
- Building a simple master-slave communication flow
- Debugging embedded hardware/software issues

---

## Project Status

Current status:

- STM32CubeMX project generated
- USART2 configured
- PA5 configured as onboard LED output
- Slave firmware implemented
- Master firmware implemented
- Basic UART PING/PONG test in progress

Next milestone:

- Implement packet-based motor command protocol with ACK/NACK and CRC.
