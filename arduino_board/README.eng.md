# Arduino RFID Interface README

This README is based only on the Arduino RFID code (`RFID_arduino.ino`) and the STM32 receiver code (`arduino.c`).

---

## 1. Role of Arduino in the System

In this project, the Arduino handles RFID card recognition, while the STM32 only receives the authentication result signal from the Arduino.

The roles are divided as follows.

| Board   | Role                                                                                     |
| ------- | ---------------------------------------------------------------------------------------- |
| Arduino | Reads the RC522 RFID card, compares the UID, and determines whether access is authorized |
| STM32   | Receives the authentication signal from Arduino and controls the door servo motor        |
| RC522   | Reads the RFID card UID                                                                  |

When an authorized RFID card is detected by the Arduino, the Arduino outputs a HIGH signal through the `D6` pin for a short period of time.
The STM32 detects this signal through the `PA4` input pin and opens the door servo motor when the authentication signal is received.

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

The Arduino code is written in `RFID_arduino.ino`, and its main functions are as follows.

* Initializes the RC522 RFID module using SPI communication
* Checks whether a new RFID card has been detected
* Reads the card UID
* Compares the UID with the registered UID list
* Outputs a HIGH pulse to the STM32 when an authorized card is detected
* Prints the UID and authentication result through the Serial Monitor

The following libraries are used.

```cpp
#include <SPI.h>
#include <MFRC522.h>
```

`SPI.h` is used for SPI communication between the Arduino and the RC522 module, and `MFRC522.h` is used to control the RC522 RFID module.

---

## 3. Pin Configuration

### Arduino Pin Map

| Arduino Pin | Connected Module | Function                                                                 |
| ----------- | ---------------- | ------------------------------------------------------------------------ |
| D10         | RC522 SDA/SS     | Slave Select signal used to select the RC522 module in SPI communication |
| D9          | RC522 RST        | Reset signal for the RC522 module                                        |
| D6          | STM32 PA4        | Output signal sent to the STM32 when RFID authentication succeeds        |
| SPI Pins    | RC522 SPI        | SPI communication for exchanging RFID data with the RC522                |

The pins are defined in the Arduino code as follows.

```cpp
#define SS_PIN          10
#define RST_PIN          9
#define STM_SIGNAL_PIN   6
```

* `SS_PIN` is connected to the SDA/SS pin of the RC522.
* `RST_PIN` is connected to the Reset pin of the RC522.
* `STM_SIGNAL_PIN` is the output pin used to send the authentication result to the STM32.

### STM32 Pin Map

| STM32 Pin | Connected Module | Function                                                         |
| --------- | ---------------- | ---------------------------------------------------------------- |
| PA4       | Arduino D6       | Receives the RFID authentication success signal from the Arduino |

In `arduino.c`, `PA4` is configured as an input pin with a pull-down resistor so that its default state remains LOW.

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

Authorized RFID cards are stored in the `authorizedCards` array as UID values.

```cpp
byte authorizedCards[][4] = {
  {0xBE, 0xAF, 0x08, 0x02}
};
```

In the current code, only the card with the UID `BE AF 08 02` is registered as an authorized card.
To add another card, add its UID value to the array.

Example:

```cpp
byte authorizedCards[][4] = {
  {0xBE, 0xAF, 0x08, 0x02},
  {0x12, 0x34, 0x56, 0x78}
};
```

The number of registered cards is automatically calculated using the array size.

```cpp
const int NUM_AUTH_CARDS = sizeof(authorizedCards) / sizeof(authorizedCards[0]);
```

---

## 5. RFID Detection Flow

In the Arduino `loop()` function, the code first checks whether a new RFID card has been detected.

```cpp
if (!rfid.PICC_IsNewCardPresent()) {
  return;
}
```

If no new card is detected, the loop exits without performing any further action.
If a card is detected, the UID is read.

```cpp
if (!rfid.PICC_ReadCardSerial()) {
  return;
}
```

When the UID is successfully read, the UID is printed to the Serial Monitor.

```cpp
Serial.print("[RFID] Card detected. UID: ");
printUID(rfid.uid.uidByte, rfid.uid.size);
Serial.println();
```

The `isAuthorized()` function then checks whether the UID belongs to a registered card.

```cpp
if (isAuthorized(rfid.uid.uidByte, rfid.uid.size)) {
  Serial.println("[RFID] Authorized card");
  sendSTM32Pulse();
} else {
  Serial.println("[RFID] Unauthorized card");
}
```

If the card is authorized, the Arduino sends a HIGH pulse to the STM32.
If the card is not authorized, only a log message is printed and no signal is sent.

---

## 6. UID Authorization Logic

The `isAuthorized()` function compares the scanned UID with the registered UID list.

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

The current code only authorizes cards with a 4-byte UID.
If the UID size is not 4 bytes, the function immediately returns `false`.

The comparison process is as follows.

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

When an authorized card is detected, the `sendSTM32Pulse()` function is executed.

```cpp
void sendSTM32Pulse(void) {
  digitalWrite(STM_SIGNAL_PIN, HIGH);
  delay(SIGNAL_PULSE_MS);
  digitalWrite(STM_SIGNAL_PIN, LOW);

  Serial.println("[STM32] D6 HIGH pulse sent");
}
```

The signal pulse width is defined by the following constant.

```cpp
const unsigned long SIGNAL_PULSE_MS = 100;
```

This means that when an authorized card is detected, the Arduino outputs HIGH on the `D6` pin for **100 ms** and then returns it to LOW.

```text
Authorized RFID
   ↓
D6 HIGH for 100 ms
   ↓
D6 LOW
```

The STM32 detects this HIGH pulse on `PA4` and triggers the door opening event.

---

## 8. STM32 Side Signal Reception

On the STM32 side, Arduino signal reception is handled in the `arduino.c` file.

```c
int Read_Arduino_Signal(void)
{
    return Macro_Check_Bit_Set(GPIOA->IDR, 4);
}
```

This function reads the input state of `PA4`.
It returns 1 when the pin is HIGH and 0 when the pin is LOW.

In the main control logic, the previous signal state and current signal state are compared to detect a rising edge.

```c
if ((rfid_signal == 1) && (prev_rfid_signal == 0)) {
    rfid_open_until = System_Tick + 10000;
}
```

In other words, the STM32 detects the moment when the 100 ms HIGH pulse from the Arduino arrives and opens the door for 10 seconds.

---

## 9. Serial Monitor Output

The Arduino prints RFID detection and authentication results to the Serial Monitor.
The serial communication speed is 115200 bps.

```cpp
Serial.begin(115200);
```

When the system starts, the following message is printed.

```text
==================================
RFID -> STM32 Signal System Start
- Authorized RFID -> D6 HIGH Pulse
==================================
```

When a card is detected, the UID and authentication result are printed.

```text
[RFID] Card detected. UID: BE AF 08 02
[RFID] Authorized card
[STM32] D6 HIGH pulse sent
```

When an unauthorized card is detected, the following message is printed.

```text
[RFID] Card detected. UID: XX XX XX XX
[RFID] Unauthorized card
```

---

## 10. Hardware Connection Notes

Since the Arduino and STM32 are different boards, voltage levels and the GND reference must be handled carefully when connecting signals.

### Common GND

The Arduino GND and STM32 GND must be connected together.

```text
Arduino GND ─── STM32 GND
```

If the GND is not shared, the STM32 may not reliably recognize the HIGH/LOW signal from the Arduino.

### Voltage Level

The digital output HIGH of Arduino UNO-based boards is generally 5 V.
Some STM32F411 pins may be 5 V tolerant, but in a real circuit, signal instability can still occur depending on voltage differences between boards, noise, and the power state of surrounding modules.

Therefore, when sending the Arduino D6 signal to STM32 PA4, it is safer to use a voltage divider circuit to reduce the 5 V signal to approximately 3.3 V.

Example:

```text
Arduino D6 ── 1kΩ ── STM32 PA4
                    │
                   2kΩ
                    │
                   GND
```

Using this structure, the Arduino 5 V HIGH signal is reduced to approximately 3.3 V at the STM32 input, enabling more stable signal transmission.

---

## 11. Full Operation Summary

```text
System start
   ↓
Initialize Arduino Serial / SPI / RC522
   ↓
Initialize STM_SIGNAL_PIN(D6) to LOW
   ↓
Wait for RFID card detection
   ↓
Card detected
   ↓
Read UID
   ↓
Compare with UIDs registered in authorizedCards[]
   ┣ Unregistered card
   │    └ Print log only
   │
   ┗ Registered card
        ↓
      Output D6 HIGH pulse for 100 ms
        ↓
      Detect HIGH pulse on STM32 PA4
        ↓
      STM32 opens the door servo motor for 10 seconds
```

---

## 12. File Summary

| File               | Description                                                                           |
| ------------------ | ------------------------------------------------------------------------------------- |
| `RFID_arduino.ino` | RC522 RFID card recognition, UID authentication, and STM32 authorization pulse output |
| `arduino.c`        | STM32 PA4 input initialization and Arduino authentication signal reading              |

---

## 13. Notes

* The current authorized card is registered with the UID `BE AF 08 02`.
* The authorization signal is output from Arduino `D6` as a 100 ms HIGH pulse.
* The STM32 detects this pulse through the `PA4` input pin.
* Arduino and STM32 must share a common GND.
* For stable signal transmission, it is recommended to connect the Arduino 5 V output to the STM32 input through a voltage divider circuit.
