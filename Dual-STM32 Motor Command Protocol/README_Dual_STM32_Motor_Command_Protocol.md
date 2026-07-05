# Dual STM32 Motor Command Protocol

A dual-microcontroller embedded project using two **STM32 Nucleo-F446RE** boards. The project implements a simple UART-based motor command protocol where one STM32 acts as a **Master Controller** and the second STM32 acts as a **Motor Slave Controller**.

The Master sends structured motor control packets containing left and right PWM values. The Slave validates each packet using a start byte and a simple XOR checksum, then responds with `ACK` or `NACK`.

This project is designed as a communication layer prototype for autonomous robotics and NXP Cup-style line follower systems.

---

## Project Goal

The goal of this project is to build a basic but realistic embedded communication protocol between two microcontrollers.

The project demonstrates:

- UART communication between two STM32 boards
- Master–Slave architecture
- Binary packet transmission
- Command-based protocol design
- Packet validation using checksum
- ACK/NACK response mechanism
- Basic abstraction for motor control commands
- Embedded C programming using STM32 HAL

---

## Hardware Used

- 2 × STM32 Nucleo-F446RE boards
- USB cable for programming
- Jumper wires
- Optional external power source for future motor control extensions

For this version, no real motors are required. The onboard LED is used as a visual indicator that a valid packet was received.

---

## Software Used

- STM32CubeMX
- STM32CubeIDE
- STM32CubeF4 firmware package
- STM32 HAL drivers

Target board:

```text
NUCLEO-F446RE
MCU: STM32F446RET6 / STM32F446RETx
Core: ARM Cortex-M4
```

---

## Project Structure

This project is split into two separate STM32CubeIDE projects:

```text
STM32_Motor_Master
STM32_Motor_Slave
```

### STM32_Motor_Master

The Master board is responsible for creating and sending motor command packets.

Main responsibilities:

- Generate left and right PWM command values
- Build a binary UART packet
- Calculate checksum
- Send the packet to the Slave
- Wait for `ACK` response
- Toggle onboard LED when a valid ACK is received

### STM32_Motor_Slave

The Slave board is responsible for receiving and validating packets.

Main responsibilities:

- Receive a binary packet over UART
- Check the start byte
- Check the command ID
- Validate left and right PWM values
- Calculate and verify checksum
- Send `ACK` if the packet is valid
- Send `NACK` if the packet is invalid
- Toggle onboard LED when a valid packet is received

---

## UART Configuration

Both STM32 boards use USART2 with the following configuration:

```text
USART: USART2
Mode: Asynchronous
Baud rate: 115200
Word length: 8 bits
Parity: None
Stop bits: 1
Mode: TX/RX
```

Pins used on the NUCLEO-F446RE:

```text
PA2 = USART2_TX = Arduino D1
PA3 = USART2_RX = Arduino D0
PA5 = LD2 onboard green LED
```

---

## Wiring

The UART connection must be crossed:

```text
MASTER PA2 / D1 / TX  -> SLAVE PA3 / D0 / RX
MASTER PA3 / D0 / RX  -> SLAVE PA2 / D1 / TX
MASTER GND            -> SLAVE GND
```

If only one USB cable is used, the Master can power the Slave for this low-current test:

```text
MASTER 5V  -> SLAVE 5V
MASTER GND -> SLAVE GND
```

Do not connect both USB power and 5V inter-board power at the same time unless you are sure your setup is safe.

If both boards are powered separately through USB, connect only:

```text
GND
TX
RX
```

---

## Communication Protocol

Each motor command packet contains 5 bytes:

```text
START | COMMAND | LEFT_PWM | RIGHT_PWM | CRC
```

Packet fields:

```text
START     = 0xA5
COMMAND   = 0x01 for SET_PWM
LEFT_PWM  = 0 to 100
RIGHT_PWM = 0 to 100
CRC       = START ^ COMMAND ^ LEFT_PWM ^ RIGHT_PWM
```

Example packet:

```text
0xA5 0x01 30 70 CRC
```

The checksum is calculated using XOR:

```c
crc = start ^ command ^ left_pwm ^ right_pwm;
```

---

## Packet Structure in C

```c
typedef struct
{
    uint8_t start;
    uint8_t command;
    uint8_t left_pwm;
    uint8_t right_pwm;
    uint8_t crc;
} motor_packet_t;
```

Constants:

```c
#define START_BYTE  0xA5
#define CMD_SET_PWM 0x01
```

---

## Slave Logic

The Slave receives a packet and validates it.

```c
HAL_UART_Receive(&huart2, (uint8_t *)&packet, sizeof(packet), HAL_MAX_DELAY);

uint8_t calculated_crc = packet.start ^ packet.command ^ packet.left_pwm ^ packet.right_pwm;

if (packet.start == START_BYTE &&
    packet.command == CMD_SET_PWM &&
    packet.crc == calculated_crc &&
    packet.left_pwm <= 100 &&
    packet.right_pwm <= 100)
{
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    HAL_UART_Transmit(&huart2, ack, sizeof(ack) - 1, HAL_MAX_DELAY);
}
else
{
    HAL_UART_Transmit(&huart2, nack, sizeof(nack) - 1, HAL_MAX_DELAY);
}
```

Expected Slave behavior:

```text
Valid packet   -> toggle LED + send ACK
Invalid packet -> send NACK
```

---

## Master Logic

The Master sends different simulated motor commands.

```c
static uint8_t state = 0;

packet.start = START_BYTE;
packet.command = CMD_SET_PWM;

if (state == 0)
{
    packet.left_pwm = 30;
    packet.right_pwm = 70;
    state = 1;
}
else if (state == 1)
{
    packet.left_pwm = 70;
    packet.right_pwm = 30;
    state = 2;
}
else
{
    packet.left_pwm = 50;
    packet.right_pwm = 50;
    state = 0;
}

packet.crc = packet.start ^ packet.command ^ packet.left_pwm ^ packet.right_pwm;

HAL_UART_Transmit(&huart2, (uint8_t *)&packet, sizeof(packet), HAL_MAX_DELAY);

if (HAL_UART_Receive(&huart2, rx_response, 5, 1000) == HAL_OK)
{
    if (rx_response[0] == 'A' &&
        rx_response[1] == 'C' &&
        rx_response[2] == 'K')
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
    }
}

HAL_Delay(1000);
```

The Master cycles through three simulated movement states:

```text
LEFT_PWM = 30, RIGHT_PWM = 70 -> simulated turn
LEFT_PWM = 70, RIGHT_PWM = 30 -> simulated opposite turn
LEFT_PWM = 50, RIGHT_PWM = 50 -> simulated forward movement
```

---

## Expected Result

When the system works correctly:

```text
The Master sends a motor command packet every second.
The Slave receives and validates the packet.
The Slave toggles its onboard LED when the packet is valid.
The Slave sends ACK back to the Master.
The Master toggles its onboard LED when ACK is received.
```

The important onboard LED is:

```text
LD2 green LED = PA5
```

LD1 is related to the ST-LINK interface and is not the main user LED for this project.

---

## Build and Flash Steps

### 1. Flash the Slave

1. Connect the Slave board to the computer using USB.
2. Open `STM32_Motor_Slave` in STM32CubeIDE.
3. Build the project.
4. Run/flash the project to the board.
5. Disconnect the Slave board after programming.

A successful flash should show:

```text
Download verified successfully
```

### 2. Flash the Master

1. Connect the Master board to the computer using USB.
2. Open `STM32_Motor_Master` in STM32CubeIDE.
3. Build the project.
4. Run/flash the project to the board.
5. Keep the Master connected to USB.

### 3. Connect the Boards

After both boards are programmed, connect:

```text
MASTER TX -> SLAVE RX
MASTER RX -> SLAVE TX
MASTER GND -> SLAVE GND
```

Power the boards according to your setup.

---

## Common Build Errors

### Wrong pointer cast in HAL_UART_Receive

Incorrect:

```c
HAL_UART_Receive(&huart2, (uint8_t)&packet, sizeof(packet), HAL_MAX_DELAY);
```

Correct:

```c
HAL_UART_Receive(&huart2, (uint8_t *)&packet, sizeof(packet), HAL_MAX_DELAY);
```

### rx_response declared incorrectly

Incorrect:

```c
uint8_t rx_response;
```

Correct:

```c
uint8_t rx_response[5];
```

### Wrong HAL delay function name

Incorrect:

```c
HAL_DELAY(1000);
```

Correct:

```c
HAL_Delay(1000);
```

---

## Troubleshooting

### Slave LED does not toggle

Possible causes:

- TX and RX are not crossed correctly
- GND is not connected between the boards
- Slave is not powered correctly
- Master is not sending packets
- USART configuration differs between boards
- Slave was not flashed with the correct firmware

### Master LED does not toggle

Possible causes:

- Slave did not receive a valid packet
- Slave did not send ACK
- Master RX wire is not connected correctly
- ACK buffer size is wrong
- UART timeout expired

### ST-LINK not detected

Possible causes:

- USB cable is not connected
- USB cable is charge-only
- Wrong board is connected
- STM32CubeProgrammer is already using ST-LINK
- Driver issue

---

## Future Improvements

This project can be extended into a more complete robotics control system:

- Add real PWM output for DC motor control
- Add motor driver support such as L298N, TB6612FNG or DRV8833
- Add direction control pins
- Add emergency stop command
- Add speed ramping
- Add timeout safety on the Slave
- Add packet sequence number
- Add stronger CRC-8 checksum
- Add telemetry from Slave to Master
- Add line sensor input
- Add PID control for line following
- Add integration with an FPGA-based sensor simulator

---

## Relevance to Autonomous Robotics

This project represents the communication layer of a distributed embedded robotic system.

In a real autonomous line-following robot:

- The Master would process sensor data and calculate movement commands.
- The Slave would control motors using PWM.
- UART packets would carry speed and direction commands.
- ACK/NACK would improve communication reliability.
- Checksum validation would reduce the risk of accepting corrupted commands.

This makes the project relevant for NXP Cup-style autonomous car development.

---

## Short Description

A dual-STM32 UART motor command protocol where a Master board sends structured PWM command packets to a Slave board. The Slave validates packets using a start byte and XOR checksum, then responds with ACK or NACK. The project demonstrates embedded UART communication, binary protocol design and basic motor control abstraction for autonomous robotics.
