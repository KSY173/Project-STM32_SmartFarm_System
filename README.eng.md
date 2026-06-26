# 🌱 STM32 Smart Farm System

![C](https://img.shields.io/badge/Language-C-A8B9CC?style=for-the-badge\&logo=c\&logoColor=black)
![STM32F411](https://img.shields.io/badge/MCU-STM32F411RE-035A91?style=for-the-badge\&logo=stmicroelectronics\&logoColor=white)
![Bare-metal](https://img.shields.io/badge/Environment-Bare--metal-black?style=for-the-badge)

This project is an **intelligent automated smart farm system** implemented in C on the STM32F411 platform in a bare-metal, register-level environment.
Beyond simply reading sensor values, the system applies practical embedded control techniques such as a Moving Average Filter, Hysteresis, Software Debouncing, and Non-blocking State Machines to improve stability and reliability.

<br>

## 📌 1. Project Overview

* **Development Environment:** STM32F411RE (Nucleo-64), GCC ARM Toolchain, Make
* **Main Components:** Arduino for RFID support, L293D motor driver, SG90 servo motor, ULN2003 stepper motor driver, flame sensor, water level sensor, light sensor, RGB/status LEDs, piezo buzzer
* **Key Features:**

  * Real-time multitasking using external interrupts (EXTI) and hardware timers (TIM)
  * Real-time PC terminal dashboard output through UART communication
  * Robust hardware design with separated power lines and noise filtering

**🎥 Demo Video:** [Watch on YouTube](https://youtu.be/-0fGDAzdwK4)

<br>

## 💡 2. Core Features

### 🚪 [1] Security Access Control System: RFID & Servo

* **Operation:**
  When the Arduino detects an authorized RFID card through the RC522 module and sends a HIGH signal, the STM32 detects the signal and smoothly opens the servo-controlled door for 10 seconds.

* **Software Design:**
  The door opens and closes smoothly by moving the servo motor by 10 degrees every 10 ms without blocking the main `while(1)` loop. This was implemented using a non-blocking control structure.

<img width="600" alt="RFID-based access control system" src="https://github.com/user-attachments/assets/c2713635-2f7c-4b8c-a968-087ef150e917" />

---

### 💧 [2] Automatic Watering and Water Level Alert System: Pump & Servo

* **Operation:**
  The water level sensor checks the remaining amount of water in the tank. If the water level is too low, the buzzer is activated and pump operation is forcibly blocked.

* **Watering Mechanism:**
  When enough water is available, the water pump is driven through the L293D motor driver at predefined intervals. At the same time, the servo motor swings the hose in a fan-shaped motion to distribute water evenly.

<img width="600" alt="Automatic watering and water level alert system" src="https://github.com/user-attachments/assets/88eb8d04-8c61-4465-8c38-0c5f40435e00" />

---

### ☀️ [3] Intelligent Light Control System: Light Sensor & Stepper Motor

* **Operation:**
  Based on the light sensor value, the auxiliary LED brightness is gradually adjusted in three levels. When the light is too strong or too weak, the stepper motor automatically opens or closes the blinds.

* **Hardware Design:**
  To prevent motor wires from tangling during blind operation, both vertical straight wiring and diagonal cross-wiring structures were applied.

<img width="600" alt="Intelligent light control system" src="https://github.com/user-attachments/assets/2a369d44-22d8-447c-900a-f34cb1cd6b16" />

---

### 🔥 [4] Fire Detection and Emergency Evacuation System: Flame Sensor

* **Operation:**
  When the flame sensor detects fire, all systems such as the pump and blinds immediately stop. The door is forced open, the red LED turns on, and the emergency buzzer alarm is activated.

* **Emergency Reset:**
  When the PC13 user button is pressed, the emergency state is cleared and the system resumes normal operation.

* **RGB LED Status Indication:**

  * Green: Normal state
  * Blue blinking: System operating
  * Red solid: Emergency state

<img width="600" alt="Fire detection and emergency evacuation system" src="https://github.com/user-attachments/assets/547d2dea-3577-4dcc-b880-22e5ca0b52fe" />

<br>

## 🛠️ 3. Troubleshooting

### 1. [Hardware] Unstable RFID Signal and Arduino Reset Issue

* **Issue:**
  The Arduino 5 V HIGH signal was directly connected to STM32 PA4, which is a 5 V tolerant pin. However, the signal became unstable and the RFID recognition performance of the Arduino dropped significantly.

* **Solution:**
  Although the STM32 pin could tolerate 5 V, a direct connection between two different boards caused current instability.
  To solve this, a **voltage divider circuit** was added using resistors to convert the 5 V signal to 3.3 V, enabling stable signal transmission.

---

### 2. [Hardware] System Reset During Motor Operation Due to Current Shortage

* **Issue:**
  When the DC motor pump and servo motor operated at the same time, the STM32 board occasionally reset. The servo motor also lost holding torque and vibrated.

* **Solution:**
  The issue was diagnosed as a brown-out condition caused by the USB power limit of approximately 500 mA.
  The logic power supply of the STM32 board and the 5 V motor power supply were separated, and an external power source was supplied to the motor VCC. The grounds were tied together as a **common GND**.
  Additionally, an L293D channel short issue was identified, and the motor control circuit was migrated to a spare channel.

---

### 3. [Software] Chattering in Light and Water Level Sensors

* **Issue:**
  The blinds and LEDs repeatedly malfunctioned due to small changes in light, shadows, and sensor noise. In addition, residual reverse flow in the hose caused the pump logic to enter an unintended repeated operation loop.

* **Solution:**
  A **Moving Average Filter** was implemented by collecting 50 samples at 100 ms intervals to reduce sensor noise.
  In addition, **Hysteresis** thresholds were applied between ON and OFF conditions to prevent unnecessary repeated motor operations and vibration.

---

### 4. [Software] False Triggering Caused by Flame Sensor Noise

* **Issue:**
  Initially, fire detection was handled using EXTI external interrupts. However, the emergency system was triggered even by very short noise signals, such as static electricity or switch noise.

* **Solution:**
  EXTI-based detection was removed, and **Software Debouncing** was implemented by polling the flame sensor pin inside a TIM4 interrupt running at a 1 ms interval.
  If the signal was interrupted even for 1 ms, the counter was reset. The system entered the emergency state only when the flame signal was continuously detected for exactly 1 second, or 1000 ms.

<br>

## 🚀 4. Future Improvements

* **IoT Dashboard Expansion:**
  Currently, sensor data is displayed on a PC terminal through UART.
  As a future improvement, wireless communication will be added by integrating modules such as ESP8266 Wi-Fi or HC-06 Bluetooth. This will allow remote monitoring and control through a web frontend or smartphone application.

<br>

## 🗂️ 5. File Structure: Flat Directory Structure for One-Command Build

```text
📦 STM32_SmartFarm_System
 ┣ 📂 System & Core
 ┃ ┣ 📜 clock.c         # System core clock configuration, PLL 96 MHz setup, and runtime initialization
 ┃ ┣ 📜 timer.c         # Hardware timer interrupt and PWM configuration wrapper
 ┃ ┗ 📜 device_driver.h # Main header for integrated module definitions and register macros
 ┃
 ┣ 📂 Application & Logic
 ┃ ┣ 📜 main.c          # Main state machine and integrated sensor/actuator control logic
 ┃ ┗ 📜 exception.c     # Flame detection debouncing using TIM4 and emergency reset using EXTI13
 ┃
 ┣ 📂 Hardware Drivers
 ┃ ┣ 📜 pump.c          # DC water pump PWM control using the L293D motor driver and TIM3
 ┃ ┣ 📜 servo.c         # Servo motor control for door operation and watering hose swing using TIM2
 ┃ ┣ 📜 step.c          # Stepper motor control for automatic blind adjustment
 ┃ ┣ 📜 indicator.c     # RGB LED, auxiliary LED, and buzzer control for system status indication
 ┃ ┣ 📜 arduino.c       # Arduino RFID signal receiver using serial/digital communication
 ┃ ┣ 📜 adc.c           # Analog data acquisition from light and water level sensors using ADC1
 ┃ ┗ 📜 uart.c          # UART communication for real-time PC terminal dashboard output
 ┃
 ┗ 📂 Build & Docs
   ┣ 📜 Makefile        # One-command build script using the GCC ARM Toolchain
   ┗ 📜 README.md       # Project documentation
```
