# STM32 SmartFarm System - Code Implementation Notes

이 문서는 프로젝트 소개용 README와 별도로, **STM32 SmartFarm System의 코드 구현 방식**을 설명하기 위한 기술 문서입니다.  
프로젝트의 전체 소개, 시연 영상, 하드웨어 사진, 트러블슈팅 내용은 메인 README에 정리하고, 본 문서에서는 `main.c`, `device_driver.h`, `exception.c`, `timer.c`, `adc.c`, `indicator.c`, `step.c`, `pump.c`, `servo.c`의 구현 구조를 중심으로 설명합니다.

---

## 1. Document Scope

현재 문서에 반영된 코드 파일은 다음과 같습니다.

| File | Role |
|---|---|
| `main.c` | 전체 상태 머신, 센서 판단, 액추에이터 통합 제어 |
| `device_driver.h` | 공통 헤더, 전역 변수 extern 선언, 모듈 함수 프로토타입 통합 관리 |
| `exception.c` | TIM4 시스템 틱, 화재 감지 디바운싱, EXTI 기반 비상 해제 |
| `timer.c` | TIM4 1 ms 반복 인터럽트 설정 |
| `adc.c` | ADC1 기반 수위 센서 및 조도 센서 값 읽기 |
| `indicator.c` | 환경 LED, RGB LED, 부저 제어 |
| `step.c` | 블라인드용 스텝모터 non-blocking 제어 |
| `pump.c` | TIM3_CH4 PWM 기반 DC 워터펌프 제어 |
| `servo.c` | TIM2_CH1/CH2 PWM 기반 도어/호스 서보모터 제어 |

`arduino.c`, `uart.c`, `clock.c`, `Makefile`은 아직 세부 코드가 반영되지 않았지만, 현재 문서만으로도 핵심 제어 로직과 주요 드라이버 구조는 설명할 수 있도록 정리했습니다.

---

## 2. Overall Software Architecture

본 시스템은 STM32F411RE를 베어메탈 환경에서 제어하며, `while(1)` 메인 루프와 타이머/외부 인터럽트를 함께 사용하는 구조로 구현되어 있습니다.

```text
System Start
   ↓
Clock / UART / Timer / GPIO / ADC Initialization
   ↓
Main Loop
   ┣ RFID signal check
   ┣ Water level ADC read
   ┣ Light sensor ADC read + moving average filter
   ┣ Door servo state update
   ┣ Water pump and hose servo update
   ┣ Blind stepper motor task update
   ┣ LED / RGB / Buzzer status update
   ┗ UART monitoring output

Interrupts
   ┣ TIM4 IRQ  : 1 ms System_Tick + fire debouncing
   ┣ EXTI12   : flame sensor first trigger
   ┗ EXTI13   : user button emergency reset
```

핵심 설계 방향은 **blocking delay를 최소화하고**, `System_Tick`을 기준으로 여러 작업을 동시에 처리하는 것입니다.  
예를 들어 문 개폐, 자동 급수, 호스 스윙, 조도 샘플링, RGB LED 점멸, 수위 부족 부저 점멸이 모두 서로를 멈추지 않고 동작하도록 구성했습니다.

---

## 3. `device_driver.h` - Common Header and Module Interface

`device_driver.h`는 프로젝트 전체에서 공통으로 사용하는 헤더 파일입니다.  
각 `.c` 파일이 개별적으로 흩어져 있어도 동일한 전역 변수, 함수 선언, 레지스터 제어 매크로를 공유할 수 있도록 연결하는 역할을 합니다.

```c
#ifndef DEVICE_DRIVER_H
#define DEVICE_DRIVER_H

#include "stm32f4xx.h"
#include "option.h"
#include "macro.h"
```

이 프로젝트는 HAL 라이브러리 기반이 아니라 STM32 레지스터를 직접 제어하는 bare-metal 방식이므로, `stm32f4xx.h`를 통해 주변장치 레지스터 구조체에 접근하고, `macro.h`의 비트 제어 매크로를 사용해 GPIO, TIM, ADC, EXTI, NVIC 설정을 수행합니다.

---

### 3.1 Shared Global Variables

여러 모듈에서 동시에 참조해야 하는 상태값은 `extern volatile` 형태로 선언되어 있습니다.

```c
extern volatile unsigned int System_Tick;
extern volatile int Emergency_Flag;
extern volatile char Uart_Data;
extern volatile int Uart_Data_In;
extern volatile int System_Mode;
```

| Variable | Purpose |
|---|---|
| `System_Tick` | TIM4 인터럽트에서 1 ms마다 증가하는 시스템 기준 시간 |
| `Emergency_Flag` | 화재 감지 후 비상 모드 진입 여부를 나타내는 플래그 |
| `Uart_Data` | UART 수신 데이터 저장 변수 |
| `Uart_Data_In` | UART 데이터 수신 여부를 나타내는 플래그 |
| `System_Mode` | 시스템 동작 모드 확장용 전역 상태 변수 |

특히 `System_Tick`과 `Emergency_Flag`는 인터럽트와 메인 루프가 함께 접근하는 값이기 때문에 `volatile`로 선언되어 있습니다.  
이를 통해 컴파일러 최적화로 인해 값이 캐싱되는 것을 방지하고, 인터럽트에서 변경된 값이 메인 루프에서도 즉시 반영되도록 구성했습니다.

---

### 3.2 Module Function Prototypes

`device_driver.h`는 각 모듈의 초기화 함수와 제어 함수를 한 곳에 모아 선언합니다.

```c
extern void Clock_Init(void);
extern void TIM4_Repeat_Interrupt_Enable(int time);

extern void Uart2_Init(int baud);
extern void ADC1_Init(void);
extern int ADC1_Read_Channel(int ch);

extern void Arduino_Comm_Init(void);
extern int Read_Arduino_Signal(void);

extern void Indicator_Init(void);
extern void Servo_Init(void);
extern void Pump_Init(void);
extern void Step_Init(void);
extern void Fire_Interrupt_Init(void);
```

이 구조 덕분에 `main.c`에서는 각 드라이버의 내부 구현을 직접 알 필요 없이, 필요한 초기화 함수와 제어 함수만 호출하여 전체 시스템을 구성할 수 있습니다.

```c
Clock_Init();
Uart2_Init(115200);
TIM4_Repeat_Interrupt_Enable(1);
Arduino_Comm_Init();
Indicator_Init();
Servo_Init();
Pump_Init();
Step_Init();
ADC1_Init();
Fire_Interrupt_Init();
```

즉, `device_driver.h`는 프로젝트의 모듈 인터페이스를 정리하는 중심 파일이며, 코드 전체의 연결성과 가독성을 높이는 역할을 합니다.

---

### 3.3 Register-Level Macro Usage

코드 전반에서 다음과 같은 매크로 호출이 반복적으로 사용됩니다.

```c
Macro_Set_Bit(RCC->AHB1ENR, 0);
Macro_Clear_Bit(EXTI->IMR, 12);
Macro_Write_Block(GPIOA->MODER, 0xF, 0xA, 0);
Macro_Check_Bit_Set(GPIOC->IDR, 12);
```

이 매크로들은 `macro.h`에 정의된 것으로 보이며, 레지스터의 특정 비트를 set/clear하거나, 특정 bit field에 값을 쓰거나, 입력 상태를 확인하는 데 사용됩니다.  
README에서는 매크로의 내부 구현보다 사용 목적을 중심으로 이해하면 됩니다.

| Macro Usage | Meaning |
|---|---|
| `Macro_Set_Bit()` | 특정 레지스터 비트를 1로 설정 |
| `Macro_Clear_Bit()` | 특정 레지스터 비트를 0으로 클리어 |
| `Macro_Write_Block()` | 여러 비트로 구성된 필드에 값을 한번에 기록 |
| `Macro_Check_Bit_Set()` | 특정 비트가 1인지 확인 |

이를 통해 복잡한 비트 연산을 직접 반복해서 작성하지 않고, GPIO 모드 설정, 타이머 활성화, EXTI 마스크 설정, ADC 변환 시작 등을 일관된 방식으로 처리합니다.

---

## 4. Pin and Peripheral Mapping

| Function | Pin / Peripheral | Code File | Description |
|---|---|---|---|
| Door Servo | `PA0 / TIM2_CH1` | `servo.c` | RFID 인증 및 비상 상황에서 출입문 개폐 |
| Hose Servo | `PA1 / TIM2_CH2` | `servo.c` | 급수 중 호스를 좌우로 스윙 |
| Water Pump PWM | `PB1 / TIM3_CH4` | `pump.c` | L293D 입력용 PWM 출력 |
| Pump Control | `PB2 / GPIO Output` | `pump.c` | 펌프 ON/OFF 제어 신호 |
| Buzzer PWM | `PB0 / TIM3_CH3` | `indicator.c` | 수위 부족 및 비상 경고음 출력 |
| Step Motor | `PB8, PB9, PB10, PB14` | `step.c` | 블라인드 개폐용 4상 출력 |
| Environment LED | `PC0, PC1, PC2` | `indicator.c` | 조도 단계 표시용 보조 LED |
| RGB LED | `PA5, PA8, PA9` | `indicator.c` | 시스템 상태 색상 표시 |
| Water Sensor | `PA6 / ADC1_CH6` | `adc.c` | 수위 센서 아날로그 입력 |
| Light Sensor | `PA7 / ADC1_CH7` | `adc.c` | 조도 센서 아날로그 입력 |
| Flame Sensor | `PC12 / EXTI12` | `exception.c` | 화재 감지 입력 |
| User Button | `PC13 / EXTI13` | `exception.c` | 비상 상태 수동 해제 |
| System Tick | `TIM4` | `timer.c`, `exception.c` | 1 ms 기준 시간 생성 |

---

## 5. `main.c` - Main Control State Machine

`main.c`는 전체 시스템의 중심 제어 파일입니다.  
초기화 단계에서 클럭, UART, TIM4, Arduino 입력, LED/부저, 서보모터, 펌프, 스텝모터, ADC, 화재 인터럽트를 순서대로 설정합니다.

```c
Clock_Init();
Uart2_Init(115200);
TIM4_Repeat_Interrupt_Enable(1);

Arduino_Comm_Init();
Indicator_Init();
Servo_Init();
Pump_Init();
Step_Init();
ADC1_Init();
Fire_Interrupt_Init();
```

메인 루프에서는 매 반복마다 RFID 신호, 수위 센서 값, 조도 센서 값을 읽습니다.

```c
int rfid_signal = Read_Arduino_Signal();
int water_level = ADC1_Read_Channel(6);
int light_level = ADC1_Read_Channel(7);
```

센서 값은 즉시 액추에이터 제어에 사용되며, 동시에 1초 주기로 UART 터미널에 출력되어 실시간 디버깅이 가능하도록 구성했습니다.

---

### 4.1 RFID-Based Door Control

RFID 인증은 Arduino에서 처리하고, STM32는 Arduino가 전달하는 디지털 신호를 읽습니다.  
`prev_rfid_signal`과 현재 `rfid_signal`을 비교하여 rising edge가 발생했을 때만 문 개방 이벤트를 실행합니다.

```c
if ((rfid_signal == 1) && (prev_rfid_signal == 0)) {
    rfid_open_until = System_Tick + 10000;
}
```

인증이 되면 `rfid_open_until` 값을 현재 시간 기준 10초 뒤로 설정합니다.  
이후 `System_Tick`이 해당 시간보다 작으면 목표 문 각도를 90도로 유지하고, 시간이 지나면 목표 각도를 0도로 되돌립니다.

```c
if (System_Tick < rfid_open_until) {
    target_door_angle = 90;
    is_active = 1;
} else {
    target_door_angle = 0;
}
```

도어 서보는 목표 각도로 한 번에 이동하지 않고, 10 ms마다 10도씩 이동합니다.  
이를 통해 서보가 갑자기 튀듯이 움직이지 않고 부드럽게 열리고 닫히도록 했습니다.

```c
if (System_Tick - last_door_move_time >= 10) {
    if (current_door_angle < target_door_angle) current_door_angle += 10;
    else if (current_door_angle > target_door_angle) current_door_angle -= 10;

    Servo_Door_Set(current_door_angle);
    last_door_move_time = System_Tick;
}
```

---

### 4.2 Emergency Priority Logic

화재가 감지되어 `Emergency_Flag`가 1이 되면, 시스템은 일반 제어보다 비상 처리를 우선합니다.  
비상 상태에서는 도어를 강제로 90도로 개방합니다.

```c
if (Emergency_Flag) {
    target_door_angle = 90;
}
```

도어 제어 이후에는 다음 코드로 일반 급수, 조도, 블라인드 제어를 건너뜁니다.

```c
if (Emergency_Flag) continue;
```

이 구조는 화재 상황에서 자동 급수나 블라인드 제어 같은 일반 기능이 비상 동작을 방해하지 않도록 하기 위한 우선순위 처리입니다.

---

### 4.3 Water Level Hysteresis and Automatic Watering

수위 센서 값은 ADC1 채널 6에서 읽습니다.  
물 부족 판단에는 단일 임계값이 아니라 두 개의 임계값을 사용했습니다.

```c
if (water_level < 1500) {
    is_water_shortage = 1;
} 
else if (water_level > 1800) {
    is_water_shortage = 0;
}
```

이 방식은 히스테리시스 구조입니다.  
수위 값이 1500 미만이면 물 부족 상태로 진입하고, 1800을 초과해야 정상 상태로 복귀합니다.  
센서 값이 경계 근처에서 흔들릴 때 펌프와 부저가 반복적으로 ON/OFF 되는 현상을 줄일 수 있습니다.

물이 부족하면 펌프를 정지하고, 부저를 500 ms 간격으로 점멸시킵니다.

```c
if (is_water_shortage == 1) {
    if (System_Tick - last_buzzer_toggle >= 500) {
        buzzer_state = !buzzer_state;
        Buzzer_Set(buzzer_state ? 2000 : 0);
        last_buzzer_toggle = System_Tick;
    }
    Pump_Set(0, 0);
    is_watering = 0;
}
```

물이 충분하면 5초 주기로 자동 급수를 시작하고, 급수는 5초 동안 유지됩니다.

```c
if (!is_watering && (System_Tick - last_watering_time >= 5000)) {
    is_watering = 1;
    water_start_time = System_Tick;
}
```

급수 중에는 펌프를 80% duty로 동작시키고, 호스 서보모터를 0도에서 50도 범위로 왕복시켜 물을 골고루 분사합니다.

```c
Pump_Set(1, 80);

if (System_Tick - last_hose_sweep >= 100) {
    hose_angle += (hose_dir * 5);
    if (hose_angle >= 50 || hose_angle <= 0) hose_dir *= -1;
    Servo_Hose_Set(hose_angle);
    last_hose_sweep = System_Tick;
}
```

---

### 4.4 Light Moving Average Filter and Blind Control

조도 센서 값은 ADC1 채널 7에서 읽습니다.  
순간적인 그림자나 외부 빛 변화로 인한 오작동을 줄이기 위해 이동 평균 필터를 적용했습니다.

```c
#define LIGHT_SAMPLE_COUNT 50
```

100 ms마다 조도 값을 샘플링하고, 최대 50개의 데이터를 평균내므로 약 5초 구간의 평균 조도 값을 사용합니다.

```c
if (System_Tick - last_light_sample_time >= 100) {
    light_sum -= light_samples[light_sample_idx];
    light_samples[light_sample_idx] = light_level;
    light_sum += light_level;

    light_sample_idx = (light_sample_idx + 1) % LIGHT_SAMPLE_COUNT;
    if (samples_filled < LIGHT_SAMPLE_COUNT) samples_filled++;

    light_avg = light_sum / samples_filled;
    last_light_sample_time = System_Tick;
}
```

환경 LED 단계도 히스테리시스 방식으로 제어됩니다.  
현재 LED 단계에 따라 다른 전환 기준을 사용하기 때문에 조도 값이 경계 근처에서 흔들려도 LED 단계가 계속 바뀌지 않습니다.

```c
if (current_led_level == 3) {
    if (light_avg > 550) current_led_level = 2;
} 
else if (current_led_level == 2) {
    if (light_avg < 450) current_led_level = 3;
    else if (light_avg > 850) current_led_level = 1;
}
```

블라인드는 조도 평균값이 1700을 초과하면 닫힘 상태, 1200 미만이면 열림 상태로 전환됩니다.

```c
if (light_avg > 1700) target_blind_state = 1;
else if (light_avg < 1200) target_blind_state = 0;
```

블라인드 상태가 실제 상태와 다를 때만 스텝모터를 구동합니다.

```c
if (current_blind_state != target_blind_state) {
    is_active = 1;
    RGB_LED_Set(3);

    if (target_blind_state == 1) Step_Move_Angle(120);
    else Step_Move_Angle(-120);

    current_blind_state = target_blind_state;
}
```

---

### 4.5 RGB System Status Logic

`is_active` 변수는 현재 시스템이 실제로 액추에이터를 동작시키는 중인지 나타냅니다.  
도어 개폐, 급수, 블라인드 동작 중에는 RGB LED를 파란색으로 점멸시켜 시스템 동작 중임을 표시합니다.

```c
if (is_active) {
    if (System_Tick - last_blue_led_toggle >= 500) {
        blue_led_state = !blue_led_state;
        RGB_LED_Set(blue_led_state ? 3 : 0);
        last_blue_led_toggle = System_Tick;
    }
} else {
    RGB_LED_Set(2);
}
```

대기 상태에서는 Green, 동작 중에는 Blue blinking, 비상 상태에서는 Red를 사용합니다.

---

## 6. `timer.c` - TIM4 1 ms System Tick

`timer.c`의 `TIM4_Repeat_Interrupt_Enable()` 함수는 TIM4를 ms 단위 반복 인터럽트 타이머로 설정합니다.

```c
void TIM4_Repeat_Interrupt_Enable(int time)
{
    Macro_Set_Bit(RCC->APB1ENR, 2);

    TIM4->PSC = 96 - 1;
    TIM4->ARR = (time * 1000) - 1;

    Macro_Set_Bit(TIM4->DIER, 0);
    NVIC->ISER[0] = (1 << 30);
    Macro_Set_Bit(TIM4->CR1, 0);
}
```

타이머 계산은 다음과 같습니다.

```text
96 MHz system clock
   ↓ PSC = 96 - 1
1 MHz timer counter = 1 µs per count
   ↓ ARR = 1 * 1000 - 1
1000 µs = 1 ms update interrupt
```

`main.c`에서 `TIM4_Repeat_Interrupt_Enable(1)`을 호출하므로 TIM4는 1 ms마다 인터럽트를 발생시키며, 이 인터럽트는 `exception.c`의 `TIM4_IRQHandler()`에서 처리됩니다.

---

## 7. `exception.c` - Fire Detection and Emergency Reset

`exception.c`는 화재 감지와 비상 해제를 인터럽트 기반으로 처리합니다.

| Signal | Pin | Interrupt | Trigger |
|---|---|---|---|
| Flame Sensor | `PC12` | `EXTI12` | Rising Edge |
| User Button | `PC13` | `EXTI13` | Falling Edge |
| System Tick | `TIM4` | `TIM4_IRQHandler` | 1 ms Update |

---

### 7.1 Fire Sensor EXTI First Trigger

화재 센서는 `PC12`에 연결되어 있으며, rising edge가 발생하면 `EXTI15_10_IRQHandler()`가 호출됩니다.  
이때 즉시 비상 상태로 들어가지 않고, 먼저 EXTI12를 비활성화한 뒤 `fire_check_start`를 1로 설정합니다.

```c
if (Emergency_Flag == 0 && fire_check_start == 0) {
    Macro_Clear_Bit(EXTI->IMR, 12);
    fire_check_start = 1;
    fire_detect_count = 0;
}
```

이 구조는 화재 센서의 짧은 노이즈로 인해 외부 인터럽트가 반복 발생하는 것을 막기 위한 1차 보호 로직입니다.

---

### 7.2 TIM4-Based Software Debouncing

`TIM4_IRQHandler()`는 1 ms마다 실행되며, `fire_check_start`가 활성화된 동안 `PC12` 입력 상태를 계속 확인합니다.

```c
if (Emergency_Flag == 0 && fire_check_start == 1) 
{
    if (Macro_Check_Bit_Set(GPIOC->IDR, 12)) {
        fire_detect_count++;

        if (fire_detect_count >= 1000) {
            Emergency_Flag = 1;
            fire_check_start = 0;

            Pump_Set(0, 0);
            Env_LED_Set(0);
            RGB_LED_Set(1);
            Buzzer_Set(1000);
        }
    } 
    else {
        fire_detect_count = 0;
        fire_check_start = 0;
        Macro_Set_Bit(EXTI->IMR, 12);
    }
}
```

화재 신호가 1000 ms 이상 연속으로 유지될 때만 `Emergency_Flag`를 1로 설정합니다.  
중간에 1 ms라도 신호가 끊기면 카운트를 초기화하고 EXTI12를 다시 활성화합니다.

```text
EXTI12 first trigger
   ↓
Disable EXTI12
   ↓
TIM4 samples PC12 every 1 ms
   ↓
PC12 HIGH for 1000 consecutive samples?
   ┣ No  → reset counter + re-enable EXTI12
   ┗ Yes → Emergency_Flag = 1
```

비상 상태 진입 시 펌프와 환경 LED를 끄고, RGB LED를 Red로 설정하며, 부저를 1000 Hz로 울립니다.

---

### 7.3 Emergency Reset with User Button

`PC13` 유저 버튼은 EXTI13 falling edge로 설정되어 있습니다.  
비상 상태에서 버튼을 누르면 `Emergency_Flag`와 화재 감지 카운터를 초기화하고, EXTI12를 다시 활성화합니다.

```c
if (Emergency_Flag == 1) 
{
    Emergency_Flag = 0;
    fire_detect_count = 0;
    fire_check_start = 0;
    Macro_Set_Bit(EXTI->IMR, 12);

    Buzzer_Set(0);
    RGB_LED_Set(2);
}
```

즉, 화재 감지 로직은 다음 순서로 동작합니다.

```text
PC12 fire signal detected
   ↓
EXTI12 rising edge interrupt
   ↓
TIM4 1 ms software debouncing
   ↓
Fire signal maintained for 1000 ms
   ↓
Emergency mode ON
   ↓
Pump OFF / Env LED OFF / RGB Red / Buzzer ON
   ↓
PC13 user button pressed
   ↓
Emergency mode OFF / RGB Green / Buzzer OFF
```

---

## 8. `adc.c` - ADC1 Sensor Input

`adc.c`는 수위 센서와 조도 센서의 아날로그 값을 읽기 위해 ADC1을 초기화합니다.

```c
void ADC1_Init(void)
{
    Macro_Set_Bit(RCC->APB2ENR, 8);
    Macro_Set_Bit(RCC->AHB1ENR, 0);

    Macro_Write_Block(GPIOA->MODER, 0xF, 0xF, 12);

    ADC1->CR2 = (1 << 0);
}
```

`PA6`, `PA7`을 아날로그 모드로 설정하고, ADC1을 켭니다.

| ADC Channel | Pin | Sensor |
|---|---|---|
| `ADC1_CH6` | `PA6` | Water Level Sensor |
| `ADC1_CH7` | `PA7` | Light Sensor |

ADC 변환은 소프트웨어 트리거 방식으로 수행됩니다.

```c
int ADC1_Read_Channel(int ch)
{
    ADC1->SQR3 = ch;
    Macro_Set_Bit(ADC1->CR2, 30);

    while(!Macro_Check_Bit_Set(ADC1->SR, 1));

    return ADC1->DR;
}
```

`SQR3`에 변환할 채널을 선택하고, `SWSTART` 비트로 변환을 시작한 뒤, `EOC` 플래그가 set되면 `DR` 값을 읽습니다.

---

## 9. `servo.c` - TIM2 Servo PWM Control

도어 서보와 급수 호스 서보는 TIM2의 두 PWM 채널을 사용합니다.

| Servo | Pin | Timer Channel | Function |
|---|---|---|---|
| Door Servo | `PA0` | `TIM2_CH1` | 출입문 개폐 |
| Hose Servo | `PA1` | `TIM2_CH2` | 급수 호스 스윙 |

`Servo_Init()`에서는 `PA0`, `PA1`을 AF1으로 설정하여 TIM2 출력에 연결합니다.

```c
Macro_Write_Block(GPIOA->MODER, 0xF, 0xA, 0);
Macro_Write_Block(GPIOA->AFR[0], 0xFF, 0x11, 0);
```

TIM2는 50 Hz PWM을 만들도록 설정되어 있습니다.

```c
TIM2->PSC = 96 - 1;
TIM2->ARR = 20000 - 1;
```

계산 과정은 다음과 같습니다.

```text
96 MHz system clock
   ↓ PSC = 96 - 1
1 MHz timer tick = 1 µs
   ↓ ARR = 20000 - 1
20 ms PWM period = 50 Hz
```

각도 제어는 CCR 값을 변경하는 방식입니다.  
0도는 약 500 µs, 180도는 약 2500 µs가 되도록 계산합니다.

```c
void Servo_Door_Set(int angle) {
    TIM2->CCR1 = 500 + (angle * 2000 / 180);
}

void Servo_Hose_Set(int angle) {
    TIM2->CCR2 = 500 + (angle * 2000 / 180);
}
```

또한 도어 서보는 필요할 때 PWM 출력을 켜고 끌 수 있도록 별도의 함수가 있습니다.

```c
void Servo_Door_Enable(void)  { Macro_Set_Bit(TIM2->CCER, 0); }
void Servo_Door_Disable(void) { Macro_Clear_Bit(TIM2->CCER, 0); }
```

---

## 10. `pump.c` - TIM3 DC Water Pump PWM Control

워터펌프는 L293D 모터 드라이버와 TIM3 PWM을 사용하여 제어합니다.

| Signal | Pin | Peripheral | Function |
|---|---|---|---|
| Pump PWM | `PB1` | `TIM3_CH4` | 펌프 속도 제어 |
| Pump Control | `PB2` | GPIO Output | 펌프 ON/OFF 제어 |

`Pump_Init()`에서는 `PB1`을 AF2로 설정하여 TIM3_CH4에 연결하고, `PB2`는 일반 출력으로 설정합니다.

```c
Macro_Write_Block(GPIOB->MODER, 0x3, 0x2, 2);
Macro_Write_Block(GPIOB->MODER, 0x3, 0x1, 4);
Macro_Write_Block(GPIOB->AFR[0], 0xF, 0x2, 4);
```

TIM3는 약 1 kHz PWM을 생성하도록 설정되어 있습니다.

```c
TIM3->PSC = 960 - 1;
TIM3->ARR = 100 - 1;
```

```text
96 MHz system clock
   ↓ PSC = 960 - 1
100 kHz timer tick = 10 µs
   ↓ ARR = 100 - 1
1 ms PWM period = 1 kHz
```

펌프 제어는 `Pump_Set()` 함수에서 처리합니다.

```c
void Pump_Set(int enable, int duty)
{
    if(enable) {
        Macro_Set_Bit(GPIOB->ODR, 2);
        TIM3->CCR4 = duty;
    } else {
        Macro_Clear_Bit(GPIOB->ODR, 2);
        TIM3->CCR4 = 0;
    }
}
```

`Pump_Set(1, 80)`은 PB2를 HIGH로 설정하고 TIM3_CH4의 duty 값을 80으로 설정합니다.  
`Pump_Set(0, 0)`은 PB2를 LOW로 만들고 PWM duty를 0으로 설정해 펌프를 정지합니다.

---

## 11. `step.c` - Non-Blocking Step Motor Control

블라인드 제어용 스텝모터는 `PB8`, `PB9`, `PB10`, `PB14` 네 개의 GPIO 출력으로 구동합니다.

```c
Macro_Write_Block(GPIOB->MODER, 0x3, 0x1, 16);
Macro_Write_Block(GPIOB->MODER, 0x3, 0x1, 18);
Macro_Write_Block(GPIOB->MODER, 0x3, 0x1, 20);
Macro_Write_Block(GPIOB->MODER, 0x3, 0x1, 28);
```

스텝 시퀀스는 8-step half-step 방식입니다.

```c
const int pattern[8][4] = {
    {1,0,0,0},{1,1,0,0},{0,1,0,0},{0,1,1,0},
    {0,0,1,0},{0,0,1,1},{0,0,0,1},{1,0,0,1}
};
```

`Step_Move_Angle()`은 입력 각도를 step 수로 변환합니다.  
코드에서는 1회전을 2048 step으로 계산합니다.

```c
int steps = (2048 * angle) / 360;
```

실제 구동은 `Step_Task()`에서 처리됩니다.  
2 ms마다 한 step씩 진행하며, 목표 step 수에 도달하면 코일 출력을 모두 끕니다.

```c
if (System_Tick - last_step_tick >= 2) 
{
    Step_Drive(step_seq_idx);
    step_seq_idx += step_dir_current;
    step_current_count++;
    last_step_tick = System_Tick;
}
```

이 방식은 non-blocking 구조입니다.  
스텝모터가 회전하는 동안에도 메인 루프는 계속 실행되므로 수위 확인, 화재 감지, UART 출력, LED 상태 갱신이 중단되지 않습니다.

---

## 12. `indicator.c` - LED and Buzzer Status Output

`indicator.c`는 사용자에게 시스템 상태를 표시하는 출력 장치를 담당합니다.

| Device | Pin / Peripheral | Function |
|---|---|---|
| Environment LED | `PC0, PC1, PC2` | 조도 단계 표시 |
| RGB Red | `PA5` | 비상 상태 표시 |
| RGB Green | `PA8` | 정상 대기 상태 표시 |
| RGB Blue | `PA9` | 동작 중 상태 표시 |
| Buzzer | `PB0 / TIM3_CH3` | 수위 부족 및 화재 경고음 |

환경 LED는 조도 단계에 따라 0개부터 3개까지 켜집니다.

```c
void Env_LED_Set(int cnt) {
    GPIOC->ODR &= ~((1<<0)|(1<<1)|(1<<2));
    if(cnt >= 1) Macro_Set_Bit(GPIOC->ODR, 0);
    if(cnt >= 2) Macro_Set_Bit(GPIOC->ODR, 1);
    if(cnt >= 3) Macro_Set_Bit(GPIOC->ODR, 2);
}
```

RGB LED는 상태값에 따라 Red, Green, Blue 중 하나를 출력합니다.

```c
void RGB_LED_Set(int color) {
    GPIOA->ODR &= ~((1<<5)|(1<<8)|(1<<9));
    if(color == 1) Macro_Set_Bit(GPIOA->ODR, 5);
    if(color == 2) Macro_Set_Bit(GPIOA->ODR, 8);
    if(color == 3) Macro_Set_Bit(GPIOA->ODR, 9);
}
```

부저는 `PB0`을 TIM3_CH3 PWM 출력으로 사용합니다.  
`freq`가 0이면 부저를 끄고, 0이 아니면 해당 주파수에 맞춰 ARR와 CCR3를 설정합니다.

```c
void Buzzer_Set(int freq) {
    if(freq == 0) {
        TIM3->CCR3 = 0;
        TIM3->ARR = 100 - 1;
    } else {
        TIM3->ARR = (100000 / freq) - 1;
        TIM3->CCR3 = TIM3->ARR / 2;
    }
}
```

주의할 점은 TIM3가 펌프 PWM과 부저 PWM에 함께 사용된다는 것입니다.  
펌프는 `TIM3_CH4`, 부저는 `TIM3_CH3`를 사용하지만, 두 채널은 같은 TIM3의 `PSC`와 `ARR`를 공유합니다.  
따라서 부저 주파수 변경 시 TIM3의 ARR가 바뀌며, 펌프 PWM 주파수에도 영향을 줄 수 있습니다. 현재 코드는 기능 검증 중심의 구조이며, 향후 안정성을 높이려면 펌프와 부저를 서로 다른 타이머에 분리하는 개선이 가능합니다.

---

## 13. Timing Summary

| Task | Period / Duration | Implementation |
|---|---:|---|
| System tick | 1 ms | TIM4 update interrupt |
| Fire debounce | 1000 ms continuous HIGH | TIM4 samples PC12 every 1 ms |
| RFID door open time | 10 s | `rfid_open_until = System_Tick + 10000` |
| Door servo movement | 10 ms / 10 degrees | Gradual angle update in `main.c` |
| UART monitoring | 1 s | `printf()` status output |
| Water shortage buzzer blink | 500 ms | Toggle `Buzzer_Set(2000)` / `Buzzer_Set(0)` |
| Watering interval | 5 s | `last_watering_time` comparison |
| Watering duration | 5 s | `water_start_time` comparison |
| Hose swing | 100 ms / 5 degrees | `Servo_Hose_Set()` update |
| Light sampling | 100 ms | Moving average buffer update |
| Light averaging window | About 5 s | 50 samples × 100 ms |
| Step motor update | 2 ms / step | `Step_Task()` non-blocking update |
| RGB blue blink | 500 ms | Active-state LED toggle |

---

## 14. Stability Techniques Used in Code

| Technique | Applied Area | Purpose |
|---|---|---|
| System tick scheduling | Overall main loop | Replace blocking delay with time-based task control |
| Rising edge detection | RFID door event | Prevent repeated door-open events while signal remains HIGH |
| Hysteresis | Water level and light control | Prevent chattering around threshold values |
| Moving average filter | Light sensor | Reduce noise and sudden brightness changes |
| Software debouncing | Flame sensor | Ignore short noise pulses and require 1 s continuous detection |
| Non-blocking motor task | Step motor | Keep main loop responsive during blind movement |
| Emergency priority flag | Fire response | Stop normal logic and force emergency behavior |
| Centralized driver header | Whole project | Share module interfaces and global state through `device_driver.h` |

---

## 15. Optional Files for Further Documentation

현재 README는 주요 제어 로직, 공통 헤더 구조, 하드웨어 드라이버 대부분을 포함합니다.  
아래 파일들은 필수는 아니지만, 추가로 반영하면 코드 설명 문서를 더 세밀하게 확장할 수 있습니다.

| Optional File | Additional Details It Can Explain |
|---|---|
| `arduino.c` | RFID 인증 신호를 STM32가 어떤 GPIO에서 어떻게 읽는지 입력 핀/신호 방식 설명 가능 |
| `uart.c` | UART2 초기화, baud rate 설정, `printf()` 리다이렉션 구조 설명 가능 |
| `clock.c` | PLL 96 MHz 시스템 클럭 설정 근거와 타이머 계산의 기준 설명 가능 |
| `Makefile` | GCC ARM Toolchain 기반 빌드 명령, 오브젝트 파일, 링커/컴파일 옵션 설명 가능 |

---

## 16. Code-Level Control Flow Summary

```text
device_driver.h
 ┣ Shared extern variables
 ┣ Driver function prototypes
 ┗ Register-level macro-based module interface

Main()
 ┣ Clock_Init()
 ┣ Uart2_Init(115200)
 ┣ TIM4_Repeat_Interrupt_Enable(1)
 ┣ Arduino_Comm_Init()
 ┣ Indicator_Init()
 ┣ Servo_Init()
 ┣ Pump_Init()
 ┣ Step_Init()
 ┣ ADC1_Init()
 ┣ Fire_Interrupt_Init()
 ┗ while(1)
     ┣ Read_Arduino_Signal()
     ┣ ADC1_Read_Channel(6)  // water
     ┣ ADC1_Read_Channel(7)  // light
     ┣ UART status print every 1 s
     ┣ RFID rising edge → door open for 10 s
     ┣ Emergency_Flag check → force door open + skip normal logic
     ┣ Water shortage hysteresis
     ┣ Pump + hose servo watering routine
     ┣ Light moving average update
     ┣ Env LED level update
     ┣ Blind target decision
     ┣ Step_Task()
     ┗ RGB status update

Interrupts
 ┣ TIM4_IRQHandler()
 ┃   ┣ System_Tick++
 ┃   ┗ Fire signal 1 ms sampling / 1000 ms debounce
 ┗ EXTI15_10_IRQHandler()
     ┣ EXTI12: fire first trigger
     ┗ EXTI13: emergency reset button
```
