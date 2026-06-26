# 🌱 STM32 Smart Farm System

[English README](./README.eng.md) 

**Built by Team: 연구하세용**

![C](https://img.shields.io/badge/Language-C-A8B9CC?style=for-the-badge&logo=c&logoColor=black)
![STM32F411](https://img.shields.io/badge/MCU-STM32F411RE-035A91?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![Bare-metal](https://img.shields.io/badge/Environment-Bare--metal-black?style=for-the-badge)

STM32F411 칩셋을 활용하여 베어메탈(Bare-metal / Register-level) 환경에서 C언어로 구현한 **'지능형 자동화 스마트팜 시스템'**입니다. 단순히 센서 값을 읽는 것을 넘어, 이동 평균 필터(Moving Average Filter), 히스테리시스(Hysteresis), 소프트웨어 디바운싱(Debouncing), Non-blocking 상태 머신 등을 적용하여 실제 상용 제품 수준의 안정성을 목표로 개발했습니다.

<br>

## 📌 1. Project Overview (프로젝트 개요)
* **개발 환경:** STM32F411RE (Nucleo-64), GCC ARM Toolchain, Make
* **주요 부품:** 아두이노(RFID 보조용), L293D(워터펌프), SG90(서보모터), ULN2003(스텝모터), 화재/수위/조도 센서, RGB/보조 LED, 피에조 부저
* **주요 특징:** * 외부 인터럽트(EXTI)와 타이머(TIM)를 활용한 실시간 멀티태스킹 처리
  * UART 통신을 이용한 PC 터미널(TeraTerm) 실시간 대시보드 출력
  * 하드웨어 전력 분리 및 노이즈 필터링이 적용된 견고한 설계

**🎥 Demo Video**: [Watch on YouTube](https://youtu.be/-0fGDAzdwK4)

<br>

## 💡 2. Core Features (핵심 기능)

### 🚪 [1] 보안 출입 통제 시스템 (RFID & Servo)
* **동작:** 아두이노에서 RC522(RFID)를 인식해 High 신호를 보내면, STM32가 이를 감지하여 서보모터(출입문)를 부드럽게 10초간 개방합니다.
* **소프트웨어:** `while(1)` 루프를 멈추지 않는 Non-blocking 방식으로, 문이 10ms당 10도씩 부드럽게 열리고 닫힙니다.
<img width="600" alt="이미지1" src="https://github.com/user-attachments/assets/c2713635-2f7c-4b8c-a968-087ef150e917" />


### 💧 [2] 자동 급수 및 수위 경보 시스템 (Pump & Servo)
* **동작:** * 수위 센서가 물탱크의 잔량을 확인하여, 물이 부족하면 부저를 울리고 펌프 작동을 강제 차단합니다.
  * 물이 충분할 경우, 지정된 주기마다 L293D 모터 드라이버를 통해 워터 펌프를 가동하고, 서보모터를 부채꼴로 회전시켜(Hose Swing) 물을 골고루 분사합니다.
<img width="600" alt="이미지2" src="https://github.com/user-attachments/assets/88eb8d04-8c61-4465-8c38-0c5f40435e00" />


### ☀️ [3] 지능형 채광 조절 시스템 (Light Sensor & Stepper)
* **동작:** 조도 센서의 값을 바탕으로 보조 LED 밝기를 3단계로 점진적 조절하며, 빛이 너무 강하거나 약할 때 스텝 모터를 구동하여 블라인드를 자동으로 개폐합니다.
* **하드웨어 설계:** 블라인드 구동 시 모터 선이 엉키지 않도록 수직(일직선)과 대각(X) 교차 설계를 적용했습니다.
<img width="600" alt="이미지3" src="https://github.com/user-attachments/assets/2a369d44-22d8-447c-900a-f34cb1cd6b16" />


### 🔥 [4] 화재 감지 및 비상 대피 시스템 (Flame Sensor)
* **동작:** 화재 센서 감지 시 모든 시스템(펌프, 블라인드)을 즉시 정지하고, 문을 강제 개방하며, 적색 LED 점등 및 비상 사이렌(Buzzer)을 울립니다. PC13(유저 버튼)을 누르면 상황이 해제되고 시스템이 재가동됩니다.
* **상태 표시(RGB LED):** 평상시(초록) / 시스템 동작 중(파랑 점멸) / 비상 상황(빨강 유지)
<img width="600" alt="이미지4" src="https://github.com/user-attachments/assets/547d2dea-3577-4dcc-b880-22e5ca0b52fe" />

<br>

## 🛠️ 3. Troubleshooting (문제 해결 과정)

1. **[Hardware] RFID 신호 불안정 및 아두이노 다운 현상**
   * **Issue:** 아두이노의 5V High 신호를 STM32의 PA4(5V Tolerant 핀)에 직접 연결했으나, 신호가 튀고 아두이노의 RFID 인식 성능이 급격히 저하됨.
   * **Solution:** 핀이 5V를 견디더라도 서로 다른 보드 간의 다이렉트 연결 시 전류 불안정이 발생함을 파악. 저항을 이용한 **전압 분배 회로(Voltage Divider)**를 구성하여 5V 신호를 3.3V로 다운시켜 안정적인 신호 전달 성공.

2. **[Hardware] 모터 구동 시 시스템 리셋 (전류 기아 상태)**
   * **Issue:** DC 모터(펌프)와 서보모터가 동시에 동작할 때 STM32 보드가 다운되거나 서보모터의 홀딩 토크가 풀리며 덜덜 떨리는 현상 발생.
   * **Solution:** USB 전력(Max 500mA) 한계로 인한 Brown-out 현상으로 진단. 로직 전원(3.3V)과 모터 전원(5V)을 완벽히 분리하고, 외부 전원(CP2102 5V 라인 및 별도 배터리)을 모터 VCC에 인가한 뒤 **GND를 공통으로 묶어(Common GND)** 해결. 추가로 L293D 채널 쇼트 문제를 파악하고 예비 채널로 마이그레이션 진행.

3. **[Software] 조도 & 수위 센서의 채터링(Chattering) 현상**
   * **Issue:** 미세한 빛의 변화나 그림자에도 블라인드 모터와 LED가 오작동을 반복함, 호스 잔류 역류로 펌프 무한 루프 발생
   * **Solution:** 100ms 간격으로 50개의 데이터를 수집하는 **이동 평균 필터(Moving Average Filter)**를 구현하여 노이즈를 1차 상쇄하고, On/Off 기준값 사이에 **히스테리시스(Hysteresis)** 임계값을 적용하여 모터의 불필요한 반복 동작(덜덜거림) 완벽 차단.

4. **[Software] 화재 센서 노이즈로 인한 오작동**
   * **Issue:** 초기 EXTI(외부 인터럽트)로 화재를 감지했으나, 주변 정전기나 스위치 조작 등 미세 노이즈(1us)에도 비상 시스템이 발동됨.
   * **Solution:** EXTI를 제거하고 TIM4(1ms 주기) 인터럽트 내부에서 핀 상태를 폴링하는 **소프트웨어 디바운싱(Software Debouncing)** 적용. 단 1ms라도 신호가 끊기면 카운트를 초기화하고, **정확히 1초(1000ms) 이상 연속 감지될 때만** 비상사태로 전환되도록 로직 고도화.

<br>

## 🚀 4. Future Improvements (향후 개선점)
* **IoT 대시보드 확장:** 현재 UART로 터미널에 출력 중인 센서 데이터를 ESP8266(Wi-Fi 모듈)이나 HC-06(Bluetooth)과 연동하여, 웹 프론트엔드 또는 스마트폰 앱에서 원격 모니터링 및 제어할 수 있도록 무선 통신 기능을 추가할 계획입니다.

<br>

## 🗂️ 5. File Structure (원클릭 빌드를 위한 Flat Directory 구조)
```text
📦 STM32_SmartFarm_System
 ┣ 📂 System & Core (시스템 핵심 설정)
 ┃ ┣ 📜 clock.c         # 시스템 코어 클럭(PLL 96MHz) 세팅 및 런타임 초기화
 ┃ ┣ 📜 timer.c         # 하드웨어 타이머(TIMx) 인터럽트 및 PWM 기본 설정 래퍼(Wrapper)
 ┃ ┗ 📜 device_driver.h # 전체 모듈 및 레지스터 매크로를 통합 관리하는 메인 헤더
 ┃
 ┣ 📂 Application & Logic (메인 제어 로직)
 ┃ ┣ 📜 main.c          # 전체 상태 머신(State Machine) 및 센서/액추에이터 통합 제어 
 ┃ ┗ 📜 exception.c     # 화재 감지 디바운싱(TIM4) 및 비상 해제(EXTI13) 인터럽트
 ┃
 ┣ 📂 Hardware Drivers (주변장치 제어)
 ┃ ┣ 📜 pump.c          # L293D 모터 드라이버 기반 DC 워터펌프 PWM 제어 (TIM3)
 ┃ ┣ 📜 servo.c         # 도어 개폐 및 급수 호스 스윙용 서보모터 제어 (TIM2)
 ┃ ┣ 📜 step.c          # 채광 조절용 블라인드 스텝 모터 제어
 ┃ ┣ 📜 indicator.c     # 시스템 상태 표시용 RGB LED, 보조 LED 및 부저 제어
 ┃ ┣ 📜 arduino.c       # 아두이노(RFID 모듈) 시리얼/디지털 통신 수신부
 ┃ ┣ 📜 adc.c           # 조도/수위 센서 아날로그 데이터 수집 (ADC1)
 ┃ ┗ 📜 uart.c          # 터미널 실시간 대시보드 출력을 위한 UART 통신
 ┃
 ┗ 📂 Build & Docs (빌드 및 문서)
   ┣ 📜 Makefile        # GCC ARM Toolchain 원클릭 통합 빌드 스크립트
   ┗ 📜 README.md       # 프로젝트 명세서
