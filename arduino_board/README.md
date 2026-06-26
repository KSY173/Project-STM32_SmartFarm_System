# Arduino RFID Interface README

[English README](./README.eng.md)

Arduino RFID 코드(`RFID_arduino.ino`)와 STM32 수신 코드(`arduino.c`)만을 기준으로 작성했습니다.

---

## 1. Role of Arduino in the System

본 프로젝트에서는 RFID 카드 인식 처리를 Arduino가 담당하고, STM32는 Arduino로부터 인증 결과 신호만 전달받습니다.

즉, 시스템은 다음과 같이 역할을 분리했습니다.

| Board | Role |
|---|---|
| Arduino | RC522 RFID 카드 인식, UID 비교, 승인 여부 판단 |
| STM32 | Arduino 승인 신호 수신, 도어 서보모터 제어 |
| RC522 | RFID 카드 UID 읽기 |

Arduino에서 승인된 RFID 카드가 감지되면, Arduino의 `D6` 핀을 짧은 시간 동안 HIGH로 출력합니다.  
STM32는 이 신호를 `PA4` 입력 핀에서 감지하고, 인증 신호가 들어온 경우 출입문 서보모터를 개방합니다.

```text
RFID Card
   ↓
RC522 RFID Module
   ↓
Arduino UID Check
   ↓
Authorized Card?
   ┣ No  → Ignore
   ┗ Yes → Arduino D6 HIGH Pulse
                 ↓
              STM32 PA4 Input
                 ↓
              Door Servo Open
```

---

## 2. Arduino Code Overview

Arduino 코드는 `RFID_arduino.ino`에 작성되어 있으며, 주요 기능은 다음과 같습니다.

- SPI 통신을 이용해 RC522 RFID 모듈 초기화
- RFID 카드가 새로 인식되었는지 확인
- 카드 UID 읽기
- 등록된 UID 목록과 비교
- 승인된 카드일 경우 STM32로 HIGH 펄스 출력
- 시리얼 모니터를 통한 UID 및 인증 결과 출력

사용한 라이브러리는 다음과 같습니다.

```cpp
#include <SPI.h>
#include <MFRC522.h>
```

`SPI.h`는 Arduino와 RC522 사이의 SPI 통신을 위해 사용하고, `MFRC522.h`는 RC522 RFID 모듈 제어를 위해 사용합니다.

---

## 3. Pin Configuration

### Arduino Pin Map

| Arduino Pin | Connected Module | Function |
|---|---|---|
| D10 | RC522 SDA/SS | SPI 통신에서 RC522 모듈을 선택하는 Slave Select 신호 |
| D9 | RC522 RST | RC522 모듈 초기화 신호 |
| D6 | STM32 PA4 | RFID 인증 성공 시 STM32로 전달하는 인증 완료 신호 출력 |
| SPI Pins | RC522 SPI | RC522와 RFID 데이터를 주고받기 위한 SPI 통신 |

Arduino 코드에서는 다음과 같이 핀을 정의합니다.

```cpp
#define SS_PIN          10
#define RST_PIN          9
#define STM_SIGNAL_PIN   6
```

- `SS_PIN`은 RC522의 SDA/SS 핀과 연결됩니다.
- `RST_PIN`은 RC522의 Reset 핀과 연결됩니다.
- `STM_SIGNAL_PIN`은 STM32로 인증 결과를 전달하는 출력 핀입니다.

### STM32 Pin Map

| STM32 Pin | Connected Module | Function |
|---|---|---|
| PA4 | Arduino D6 | 아두이노에서 출력한 RFID 인증 완료 신호를 STM32가 입력으로 수신|

STM32의 `arduino.c`에서는 `PA4`를 입력 모드로 설정하고, pull-down을 적용하여 기본 상태가 LOW가 되도록 구성했습니다.

```c
void Arduino_Comm_Init(void)
{
    Macro_Set_Bit(RCC->AHB1ENR, 0); 
    Macro_Write_Block(GPIOA->MODER, 0x3, 0x0, 8); 
    Macro_Write_Block(GPIOA->PUPDR, 0x3, 0x2, 8); 
}
```

---

## 4. Authorized Card UID

승인된 RFID 카드는 `authorizedCards` 배열에 UID 형태로 저장됩니다.

```cpp
byte authorizedCards[][4] = {
  {0xBE, 0xAF, 0x08, 0x02}
};
```

현재 코드에서는 UID가 `BE AF 08 02`인 카드만 승인 카드로 등록되어 있습니다.  
다른 카드를 추가하려면 배열에 UID 값을 추가하면 됩니다.

예시:

```cpp
byte authorizedCards[][4] = {
  {0xBE, 0xAF, 0x08, 0x02},
  {0x12, 0x34, 0x56, 0x78}
};
```

등록된 카드 수는 배열 크기를 이용해 자동으로 계산됩니다.

```cpp
const int NUM_AUTH_CARDS = sizeof(authorizedCards) / sizeof(authorizedCards[0]);
```

---

## 5. RFID Detection Flow

Arduino의 `loop()` 함수에서는 먼저 새 RFID 카드가 감지되었는지 확인합니다.

```cpp
if (!rfid.PICC_IsNewCardPresent()) {
  return;
}
```

새 카드가 없으면 아무 동작도 하지 않고 loop를 빠져나갑니다.  
카드가 감지되면 UID를 읽습니다.

```cpp
if (!rfid.PICC_ReadCardSerial()) {
  return;
}
```

UID 읽기에 성공하면 시리얼 모니터에 UID를 출력합니다.

```cpp
Serial.print("[RFID] Card detected. UID: ");
printUID(rfid.uid.uidByte, rfid.uid.size);
Serial.println();
```

이후 `isAuthorized()` 함수를 통해 UID가 등록된 카드인지 확인합니다.

```cpp
if (isAuthorized(rfid.uid.uidByte, rfid.uid.size)) {
  Serial.println("[RFID] Authorized card");
  sendSTM32Pulse();
} else {
  Serial.println("[RFID] Unauthorized card");
}
```

승인된 카드라면 STM32로 HIGH 펄스를 보내고, 승인되지 않은 카드라면 로그만 출력하고 별도 신호는 보내지 않습니다.

---

## 6. UID Authorization Logic

`isAuthorized()` 함수는 읽어온 UID와 등록된 UID 목록을 비교합니다.

```cpp
bool isAuthorized(byte *uid, byte uidSize) {
  if (uidSize != 4) {
    return false;
  }

  for (int i = 0; i < NUM_AUTH_CARDS; i++) {
    bool match = true;

    for (int j = 0; j < 4; j++) {
      if (uid[j] != authorizedCards[i][j]) {
        match = false;
        break;
      }
    }

    if (match) {
      return true;
    }
  }

  return false;
}
```

현재 코드는 4바이트 UID 카드만 승인 대상으로 처리합니다.  
UID 크기가 4바이트가 아니면 즉시 `false`를 반환합니다.

비교 과정은 다음과 같습니다.

```text
Read Card UID
   ↓
UID size == 4?
   ┣ No  → Unauthorized
   ┗ Yes → Compare with authorizedCards[]
              ↓
           All bytes match?
              ┣ No  → Unauthorized
              ┗ Yes → Authorized
```

---

## 7. STM32 Signal Pulse

승인된 카드가 인식되면 `sendSTM32Pulse()` 함수가 실행됩니다.

```cpp
void sendSTM32Pulse(void) {
  digitalWrite(STM_SIGNAL_PIN, HIGH);
  delay(SIGNAL_PULSE_MS);
  digitalWrite(STM_SIGNAL_PIN, LOW);

  Serial.println("[STM32] D6 HIGH pulse sent");
}
```

신호 펄스 폭은 다음 상수로 정의되어 있습니다.

```cpp
const unsigned long SIGNAL_PULSE_MS = 100;
```

즉, Arduino는 승인된 카드가 인식되면 `D6` 핀을 **100 ms 동안 HIGH**로 출력한 뒤 다시 LOW로 내립니다.

```text
Authorized RFID
   ↓
D6 HIGH for 100 ms
   ↓
D6 LOW
```

STM32는 이 HIGH 펄스를 `PA4`에서 감지하여 도어 개방 이벤트를 발생시킵니다.

---

## 8. STM32 Side Signal Reception

STM32에서는 `arduino.c` 파일에서 Arduino 신호 수신부를 처리합니다.

```c
int Read_Arduino_Signal(void)
{
    return Macro_Check_Bit_Set(GPIOA->IDR, 4);
}
```

이 함수는 `PA4` 입력 상태를 읽어 HIGH이면 1, LOW이면 0을 반환합니다.

메인 제어 로직에서는 이전 신호 상태와 현재 신호 상태를 비교하여 rising edge를 감지합니다.

```c
if ((rfid_signal == 1) && (prev_rfid_signal == 0)) {
    rfid_open_until = System_Tick + 10000;
}
```

즉, Arduino의 100 ms HIGH 펄스가 들어오는 순간을 감지해 도어를 10초 동안 열도록 구성합니다.

---

## 9. Serial Monitor Output

Arduino는 RFID 인식 및 인증 결과를 시리얼 모니터에 출력합니다.  
시리얼 통신 속도는 115200 bps입니다.

```cpp
Serial.begin(115200);
```

시작 시 다음 메시지가 출력됩니다.

```text
==================================
RFID -> STM32 Signal System Start
- Authorized RFID -> D6 HIGH Pulse
==================================
```

카드가 감지되면 UID와 인증 결과가 출력됩니다.

```text
[RFID] Card detected. UID: BE AF 08 02
[RFID] Authorized card
[STM32] D6 HIGH pulse sent
```

승인되지 않은 카드가 감지되면 다음과 같이 출력됩니다.

```text
[RFID] Card detected. UID: XX XX XX XX
[RFID] Unauthorized card
```

---

## 10. Hardware Connection Notes

Arduino와 STM32는 서로 다른 보드이므로 신호 연결 시 전압 레벨과 GND 기준을 반드시 맞춰야 합니다.

### Common GND

Arduino의 GND와 STM32의 GND는 반드시 공통으로 연결해야 합니다.

```text
Arduino GND ─── STM32 GND
```

GND가 공통으로 연결되지 않으면 STM32가 Arduino의 HIGH/LOW 신호를 안정적으로 인식하지 못할 수 있습니다.

### Voltage Level

Arduino UNO 계열의 디지털 출력 HIGH는 일반적으로 5V입니다.  
STM32F411의 일부 핀은 5V tolerant일 수 있지만, 실제 회로에서는 보드 간 전위차, 노이즈, 주변 모듈 전원 상태에 따라 신호가 불안정해질 수 있습니다.

따라서 Arduino D6 신호를 STM32 PA4로 전달할 때는 5V를 3.3V 수준으로 낮추는 전압 분배 회로를 사용하는 것이 안전합니다.

예시:

```text
Arduino D6 ── 1kΩ ── STM32 PA4
                    │
                   2kΩ
                    │
                   GND
```

이 구조를 사용하면 Arduino의 5V HIGH 신호가 STM32 입력 기준으로 약 3.3V 수준으로 낮아져 더 안정적인 신호 전달이 가능합니다.

---

## 11. Full Operation Summary

```text
시스템 시작
   ↓
Arduino Serial / SPI / RC522 초기화
   ↓
STM_SIGNAL_PIN(D6) LOW 초기화
   ↓
RFID 카드 감지 대기
   ↓
카드 감지
   ↓
UID 읽
   ↓
authorizedCards[]에 등록된 UID와 비교
   ┣ 미등록 카드
   │    └ 로그만 출력
   │
   ┗ 등록 카드
        ↓
      D6 HIGH 100 ms Pulse 출력
        ↓
      STM32 PA4에서 HIGH pulse 감지
        ↓
      STM32가 도어 서보모터 10초 동안 개방
```

---

## 12. File Summary

| File | Description |
|---|---|
| `RFID_arduino.ino` | RC522 RFID 카드 인식, UID 인증, STM32 승인 펄스 출력 |
| `arduino.c` | STM32 PA4 입력 초기화 및 Arduino 인증 신호 읽기 |

---

## 13. Notes

- 현재 승인 카드는 `BE AF 08 02` UID로 등록되어 있습니다.
- 승인 신호는 Arduino `D6`에서 100 ms HIGH 펄스로 출력됩니다.
- STM32는 `PA4` 입력에서 해당 펄스를 감지합니다.
- Arduino와 STM32는 반드시 GND를 공통으로 연결해야 합니다.
- 안정적인 신호 전달을 위해 Arduino 5V 출력은 전압 분배 회로를 통해 STM32 입력에 연결하는 것을 권장합니다.
