# ESP32 ESP-NOW Bidirectional Wireless Network

> Real-time bidirectional wireless communication between two ESP32-S3 microcontrollers using ESP-NOW — no router, no inter-node wiring.

[![Platform](https://img.shields.io/badge/platform-ESP32--S3-blue)](https://www.espressif.com/en/products/socs/esp32-s3)
[![Protocol](https://img.shields.io/badge/protocol-ESP--NOW-teal)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
[![Cost](https://img.shields.io/badge/total%20cost-~£45-green)]()
[![License](https://img.shields.io/badge/license-MIT-purple)]()

---



---

## Overview

This project demonstrates a **two-node ESP32-S3 wireless network** where both nodes communicate bidirectionally via the **ESP-NOW protocol**. One node collects environmental data and controls a servo motor. The other displays that data on a TFT screen and lets the user send commands back via a 3-button menu.

A key aim was to evaluate whether **wireless communication could reduce design costs** by eliminating inter-node copper wiring compared to wired alternatives like CAN bus.

---

## System Architecture
──────────┐ ┌─────────────────────────┐
│ NODE 1 │ │ NODE 2 │
│ Sensor Node │ ─── SensorData ──▶│ Control Hub │
│ ESP32-S3-N16R8 │ │ ESP32-S3-N16R8 │
│ │◀── CommandData ─── │ │
│ • DHT11 (temp/humid) │ │ • ST7789 TFT display │
│ • LDR (light level) │ ESP-NOW │ • 3-button menu │
│ • DS3231 RTC (time) │ 2.4 GHz │ • State machine UI │
│ • SG90 servo │ ~2s cycle │ • Partial redraw │
│ │ │ │
│ MAC: 94:A9:90:D2:F0:98 │ │ MAC: 94:A9:90:D1:DC:20 │


---

## Features

- **Bidirectional ESP-NOW communication** — no Wi-Fi router required
- **Live environmental monitoring** — temperature, humidity, light level, time
- **Time-aware light status** — day / evening / night thresholds via DS3231 RTC
- **Servo auto-control** — triggers when temperature or humidity exceeds user-set thresholds
- **Manual servo override** — user can control servo directly from the menu
- **Flicker-free TFT display** — partial redraw technique (only changed fields redrawn)
- **3-button menu system** — finite state machine: MAIN → MENU → SET_TEMP / SET_HUMID / SERVO
- **Software debounce** — eliminates false triggers (up to 1,258 per press without it)

---

## Hardware

### Component List

| Component | Specification | Qty | Approx. Cost |
|---|---|---|---|
| ESP32-S3-N16R8 Dev Module | 16MB Flash, 8MB OPI PSRAM, 240MHz | 2 | ~£9.99 each |
| DHT11 Sensor | Temp + humidity, ±2°C, ±5% RH | 1 | ~£4.99 |
| DS3231 RTC Module | I2C, ±2ppm accuracy | 1 | ~£3.00 |
| SG90 Micro Servo | 180°, PWM | 1 | ~£2.50 |
| ST7789 TFT Display | 2.0" IPS, 240×320, SPI | 1 | ~£6.99 |
| LDR (GL5528) | Light dependent resistor | 1 | ~£0.50 |
| Tactile push buttons | Normally open, 6×6mm | 3 | ~£0.30 |
| 10kΩ resistor | Pull-down for LDR | 1 | ~£0.05 |
| Breadboard + jumper wires | 830-point + 120pc kit | 1 set | ~£6.99 |
| **Total** | | | **~£45** |

---

### Pin Assignments

**Node 1 — Sensor Node**

| Peripheral | GPIO |
|---|---|
| DHT11 Data | GPIO 4 |
| LDR (ADC) | GPIO 5 |
| Servo Signal | GPIO 21 |
| DS3231 SDA | GPIO 8 |
| DS3231 SCL | GPIO 9 |

**Node 2 — Control Hub**

| Peripheral | GPIO |
|---|---|
| TFT MOSI | GPIO 11 |
| TFT CLK | GPIO 12 |
| TFT CS | GPIO 10 |
| TFT DC | GPIO 5 |
| TFT RST | GPIO 4 |
| Button UP | GPIO 6 |
| Button DOWN | GPIO 7 |
| Button ENTER | GPIO 8 |

---

## Software

### Libraries Used

| Library | Purpose |
|---|---|
| `esp_now.h` | ESP-NOW peer-to-peer communication |
| `WiFi.h` | Wi-Fi init required for ESP-NOW |
| `DHT.h` | DHT11 sensor reading |
| `RTClib.h` | DS3231 RTC interface |
| `Adafruit_ST7789.h` | TFT display driver |
| `Adafruit_GFX.h` | Graphics primitives |
| `ESP32Servo.h` | Servo PWM control |

### Data Structs

```cpp
// Node 1 → Node 2
struct SensorData {
  float temperature;
  float humidity;
  int lightLevel;
  char lightStatus[15];
  int hour;
  int minute;
  bool servoOpen;
};

// Node 2 → Node 1
struct CommandData {
  int tempThreshold;
  int humidThreshold;
  bool manualServo;
};
```

---

## Key Engineering Challenges

| Challenge | Solution |
|---|---|
| ESP-NOW "Peer interface is invalid" error | `memset()` peer struct to zero + set `ifidx = WIFI_IF_STA` before registration |
| 1,258 false button triggers per press | Custom debounce: 50ms stable LOW check + wait-for-release loop |
| TFT screen flicker on every update | Partial redraw — store previous values, only redraw changed fields |
| Display blank until first value change | `firstDataReceived` flag triggers full initial draw on first packet |
| Servo not responding to manual command | Isolated component test confirmed issue; systematic diagnostics resolved it |

---

## Arduino IDE Setup

1. Install **ESP32 board package** via Boards Manager (`https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`)
2. Select board: `ESP32S3 Dev Module`
3. Settings:
   - USB CDC On Boot: `Enabled`
   - Flash Size: `16MB`
   - PSRAM: `OPI PSRAM`
   - Partition Scheme: `16M Flash (3MB APP/9MB FATFS)`
   - Upload Mode: `UART0/Hardware CDC`
4. Install libraries via Library Manager:
   - `DHT sensor library` by Adafruit
   - `RTClib` by Adafruit
   - `Adafruit ST7735 and ST7789 Library`
   - `Adafruit GFX Library`
   - `ESP32Servo`
5. Upload `node1_sensor/node1_sensor.ino` to Node 1
6. Upload `node2_hub/node2_hub.ino` to Node 2

> **Note:** Hold BOOT then press RST to enter upload mode if the upload fails. After uploading to Node 2, press the physical RST button to initialise the TFT display.

---

## File Structure

esp32-espnow-bidirectional-network/
├── node1_sensor/
│ └── node1_sensor.ino # Sensor node firmware
├── node2_hub/
│ └── node2_hub.ino # Control hub firmware
├── report/
│ └── ESP32_ESP-NOW_Engineering_Report.docx
└── README.md

---

## Results

- ✅ Stable bidirectional ESP-NOW communication confirmed — no packet loss at bench range
- ✅ SensorData transmitted every 2 seconds, received and displayed correctly
- ✅ CommandData transmitted on user input, servo responds within one loop cycle
- ✅ TFT display flicker-free with partial redraw
- ✅ Button menu navigates and adjusts thresholds correctly
- ✅ Servo responds to both automatic threshold triggers and manual override
- ✅ Total build cost ~£45 — significantly lower than equivalent wired CAN bus implementation

---

## Future Improvements

- Extend to 3+ nodes for true mesh networking
- Add ESP-NOW encryption for secure transmission
- Data logging to flash via LittleFS
- Replace DHT11 with BME280 for higher accuracy
- Web dashboard over Wi-Fi alongside ESP-NOW

---

## Author

**Emmanuel**
BEng Electrical & Electronic Engineering — University of the West of England, Bristol
[LinkedIn](https://www.linkedin.com/in/PABLLO1127) · [GitHub](https://github.com/PABLLO1127)

---

## License

MIT License — free to use, modify, and distribute with attribution.
