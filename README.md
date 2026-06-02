# esphome-dewenwils-ptw04
ESPHome firmware for the Dewenwils PTW04 240VAC Smart Timer Box — replaces Tuya cloud / Dewenwils app with local Home Assistant control via BK7231N (LibreTiny)

# Dewenwils PTW04 — ESPHome Firmware

> Replace the stock Tuya firmware on the Dewenwils PTW04 Smart Timer Box with ESPHome, enabling local control via Home Assistant without any cloud dependency.

<img width="630" height="630" alt="image" src="https://github.com/user-attachments/assets/109f06f6-71f5-4f1f-b279-bf0fe2caec88" />


---

## Table of Contents

- [Product Overview](#product-overview)
- [Hardware Details](#hardware-details)
- [GPIO Map](#gpio-map)
- [Flashing Prerequisites](#flashing-prerequisites)
- [Flashing Procedure](#flashing-procedure)
- [ESPHome Configuration](#esphome-configuration)
- [Home Assistant Integration](#home-assistant-integration)
- [Behavior & Features](#behavior--features)
- [Credits](#credits)

---

## Product Overview

| Field | Value |
|-------|-------|
| **Brand** | Dewenwils |
| **Model** | PTW04 |
| **Type** | Smart Timer Box / Load Controller |
| **Input** | 240VAC 60Hz |
| **Output** | 40A Resistive 240VAC / 3HP 240VAC |
| **Certifications** | ETL Listed, FCC ID: 2A4G9-016, IC: 26865-014 |
| **App** | SmartLife (Tuya) |
| **PCB Reference** | RT-PTW01-1.0 |
| **PCB Date** | 20240523 |

The PTW04 is a WiFi-enabled load controller designed for 240VAC applications such as pool pumps, heaters, and motors. It uses two relays wired in series to switch both L1 and L2 lines of a 240VAC circuit simultaneously. A third relay footprint (NO3/K1) is present on the power board but not populated.

<img width="1574" height="2470" alt="20260510_091451" src="https://github.com/user-attachments/assets/11db5240-16ff-406c-8d3b-f143fe9042e6" />

---

## Hardware Details

The device consists of two PCBs connected via a flat ribbon cable:

### Power Board (PTW01-POWER)

Front:
<img width="2470" height="1534" alt="20260505_201407" src="https://github.com/user-attachments/assets/a908c2b9-7283-4c1a-8111-50d8bad2f32b" />

Back:
<img width="2864" height="1718" alt="20260505_201353" src="https://github.com/user-attachments/assets/56da2ae5-d380-48c0-ac8a-105fbb2b4ef5" />


- Handles mains voltage (240VAC)
- Contains 2 populated **bistable (latching) relays** — Meishuo C-S-105-A-L1 (NO1/L1 and NO2/L2) + 1 empty footprint (NO3)
- Terminal block connections: `NO3 | N | L2 | NO2 | L2 | L1 | NO1 | L1`
- AC/DC power supply section providing 3.3V and 12V rails to the control board via flat ribbon cable
- Relay coils driven at 12V via NPN transistor drivers (U4/U5/U6) on the control board

### Control Board (RT-PTW01-1.0)

Front:
<img width="1276" height="2150" alt="20260505_201524" src="https://github.com/user-attachments/assets/2b68837c-a1c7-48dc-9e16-5ac28667e15c" />

Back:
<img width="1098" height="1952" alt="20260505_201514" src="https://github.com/user-attachments/assets/69a052b4-4df9-4de2-beb4-bcdacc41eeef" />



| Component | Description |
|-----------|-------------|
| **U1** | Beken BK7231N — WiFi/BT SoC (LibreTiny compatible) |
| **Y1** | Crystal YL260 Yx4H |
| **U7** | XJS 5350Z — 5V LDO regulator |
| **U4/U5/U6** | 3111S JRONK — Dual NPN transistors (SOT-23-6), relay drivers |
| **SW1** | Push button (ON/OFF + reset) |
| **D1** | Power LED — controlled by P22 on BK7231N |
| **D2** | Load LED (red) — indicates relay state, controlled by P21 |
| **D3** | WiFi LED (blue) — ESPHome status_led, controlled by P23 |
| **J5** | Output connector: `VIN, OUT1+, OUT1-, OUT2+, OUT2-, OUT3+, OUT3-, GND` |

#### UART Test Points (clearly silkscreened on PCB)

| Pad | Function |
|-----|----------|
| `GND` | Ground |
| `U1TX` | UART1 TX (connect to RX on USB-TTL adapter) |
| `U1RX` | UART1 RX (connect to TX on USB-TTL adapter) |
| `3V3` | 3.3V power input |
| `U2TX` | UART2 TX (reserved — do not use) |
| `U2RX` | UART2 RX (reserved — do not use) |
| `TEST` | Likely CEN (chip enable / reset) |

---

## GPIO Map

All GPIOs confirmed through empirical testing after firmware dump analysis.

| GPIO | Direction | Function | Notes |
|------|-----------|----------|-------|
| **P24** | OUTPUT | Relay L1 SET | Active LOW — latches relay L1 ON via NPN transistor |
| **P26** | OUTPUT | Relay L1 RESET | Active LOW — latches relay L1 OFF via NPN transistor |
| **P17** | OUTPUT | Relay L2 SET | Active LOW — latches relay L2 ON via NPN transistor |
| **P15** | OUTPUT | Relay L2 RESET | Active LOW — latches relay L2 OFF via NPN transistor |
| **P21** | OUTPUT | LED Load (D2 red) | Active HIGH |
| **P22** | OUTPUT | LED Power (D1) | Active HIGH — always ON when powered |
| **P23** | OUTPUT | LED WiFi (D3 blue) | Active HIGH — ESPHome status_led |
| **P28** | INPUT | Button SW1 | Active LOW — INPUT_PULLUP |

### Reserved / Unusable Pins (generic-bk7231n-qfn32-tuya)

| GPIO | Reserved For |
|------|-------------|
| P2, P3, P4, P5 | Internal use / strapping pins |
| P10, P11 | UART1 (flash / TuyaMCU) |
| P25, P27 | Reserved by LibreTiny |

### Relay Driver Circuit

The BK7231N GPIOs cannot directly drive relay coils (which require ~50-100mA at 12V). Each relay is driven through a dual NPN transistor (3111S, SOT-23-6):

```
BK7231N GPIO (active LOW)
  → Series resistor
    → NPN Base
      → NPN Collector → Relay coil (12V)
      → NPN Emitter  → GND
```

When the GPIO goes LOW, the transistor saturates and the relay coil is energized. This is why all relay outputs use `inverted: true` in ESPHome.

> **Important:** The relays on the PTW04 are **bistable (latching)** — they maintain their state without continuous current. A short pulse (100ms) on the SET line latches them ON; a short pulse on the RESET line latches them OFF. This is why the ESPHome configuration uses timed impulses rather than sustained GPIO levels.

---

## Flashing Prerequisites

### Tools Required

- USB-TTL UART adapter (CH340, CP2102, or similar) — **3.3V logic level**
- USB extension cable (highly recommended for easier manipulation)
- 4× dupont female-to-female jumper wires
- **BDM frame + 4 BDM probes** (spring-loaded pogo pins) — used in this project for reliable contact on the silkscreened pads without soldering

<img width="894" height="697" alt="image" src="https://github.com/user-attachments/assets/b7255491-b908-4c03-bb6c-868a3decbbf8" />
<img width="604" height="634" alt="image" src="https://github.com/user-attachments/assets/fd7b81fd-cc33-4dd6-8430-518fb145401a" />
<img width="570" height="313" alt="image" src="https://github.com/user-attachments/assets/6ba7ab21-586f-4133-9e20-ed69648feef3" />



> **Tip:** A BDM frame holds the 4 probes (GND, TX, RX, 3V3) in precise alignment over the pads. Combined with a USB extension cable, this allows comfortable one-handed manipulation while keeping the other hand free to trigger the boot mode.

### Software Required

- [ltchiptool](https://github.com/libretiny-eu/ltchiptool) — BK7231N flash tool
- [ESPHome](https://esphome.io) — firmware framework
- [Home Assistant](https://www.home-assistant.io) — optional but recommended

---

## Flashing Procedure

> ⚠️ **Safety first:** Always disconnect the device from 240VAC mains before opening or connecting any test equipment. Power the control board exclusively from your USB-TTL adapter during flashing. Never flash with mains voltage applied.

### Step 1 — Disconnect the Control Board

1. Open the PTW04 enclosure
2. Disconnect the flat ribbon cable between the power board and the control board
3. Work only with the control board (RT-PTW01-1.0) — the green PCB with the BK7231N

### Step 2 — Connect the USB-TTL Adapter

Using the BDM frame with 4 pogo probes, position them on the silkscreened pads:

| Control Board Pad | USB-TTL Adapter Pin |
|-------------------|---------------------|
| `GND` | GND |
| `U1TX` | **RX** (cross-connect) |
| `U1RX` | **TX** (cross-connect) |
| `3V3` | 3.3V |

<img width="1848" height="4000" alt="20260505_155049" src="https://github.com/user-attachments/assets/910271d7-f230-4649-be68-2fa0498147f0" />
<img width="1848" height="2172" alt="20260505_155101" src="https://github.com/user-attachments/assets/af414d06-7e60-456f-a5e5-dc0ac0f692a5" />


> **Note:** Do not connect to U2TX/U2RX — these are UART2 (logger) and correspond to the relay GPIOs P26/P27.

### Step 3 — Backup the Original Firmware

Always backup before flashing. Open ltchiptool, select **BK7231N** and perform a full flash read:

```
ltchiptool flash read -d /dev/ttyUSB0 backup_ptw04_original.bin
```

Keep this backup file in a safe location. It contains the original Tuya firmware and can be used to restore the device to factory state.

### Step 4 — Enter Download Mode (No CEN Pin Required)

The `TEST` pad (CEN) might be used to trigger download mode (untested), but it is not strictly necessary. The following method was used successfully on this device:

1. Launch ltchiptool and start the chip detection / flash operation
2. While ltchiptool is waiting for the chip to respond, **briefly touch a jumper wire from GND to the GND pad repeatedly** (tap GND to GND 2–3 times in quick succession during power-up)
3. This triggers a reset that causes the BK7231N to enter UART download mode
4. ltchiptool will detect the chip and proceed, at this stage, keep de GND connected to the GND pad.

> **Tip:** This may require several attempts. The timing is critical — the GND tap must happen within the first ~100ms of power-on. Using a USB extension cable makes it much easier to manipulate the board while keeping the probes in contact.


### Step 5 — Flash ESPHome Firmware

1. In ESPHome dashboard, create a new device using the provided YAML configuration in this repo. Make sure to change the WiFi, OTA and hotspot credentiels to fit your needs.
2. Click **Install** → **Manual download**, ESPHome will compiles the firmware.
<img width="624" height="497" alt="image" src="https://github.com/user-attachments/assets/dd984729-c4c5-4577-9f38-9ec5b28e0642" />

3.  → select **UF2 Package** and downloads the .uf2 firmware
<img width="461" height="489" alt="image" src="https://github.com/user-attachments/assets/be595fa5-0c98-4783-83e8-d5e7da45dcee" />

5. In ltchiptool, select the downloaded `.uf2` and flash it to the device, use the same method as steps 4.2 to trigger download mode if the control board exited download mode (if the board was disconnected between steps 4 and 5).



---

## ESPHome Configuration

> See [dewenwils-ptw04.yaml](dewenwils-ptw04.yaml) for the complete configuration.

### Key Configuration Decisions

| Setting | Value | Reason |
|---------|-------|--------|
| `baud_rate: 0` | 0 | Disables UART2 logger, freeing P26/P27 for relay GPIOs |
| `restore_mode` | `RESTORE_DEFAULT_OFF` | Actual power-on behavior managed by the select entity |
| `on_boot priority` | -100 | Ensures all components are fully initialized before boot actions run |
| Relay logic | Bistable impulse | 100ms SET/RESET pulses — relays hold state without continuous power |
| Relay pins | `inverted: true` | Relay drivers are active LOW (NPN transistor logic) |

---

## Home Assistant Integration

After flashing, power the control board via the USB-TTL adapter and wait for it to connect to WiFi. Home Assistant will automatically discover the device via mDNS.

**Discovered entities:**

| Entity | Type | Description |
|--------|------|-------------|
| Load | Switch | Main load control (both relays simultaneously) |
| Power On Behavior | Select | Configures load state after power cycle — Always OFF / Restore last state / Always ON |
| Button | Binary Sensor | Physical button state |
| WiFi Signal | Sensor | WiFi signal strength in % |
| IP Address | Text Sensor | Current IP address |
| SSID | Text Sensor | Connected WiFi network |
| Uptime | Sensor | Device uptime |
| Restart | Button | Remote restart |

---

## Behavior & Features

### LED Indicators

| LED | Color | GPIO | Behavior |
|-----|-------|------|----------|
| D1 Power | Green | P22 (active HIGH) | ON whenever the device is powered |
| D2 Load | Red | P21 (active HIGH) | ON when the load (relays) is active |
| D3 WiFi | Blue | P23 (active HIGH) | ESPHome status LED — see below |

**WiFi LED (D3) behavior with ESPHome status_led:**

| D3 State | Meaning |
|----------|---------|
| Blinking fast | No WiFi connection / booting |
| Blinking slow | WiFi connected, API disconnected |
| OFF | WiFi + Home Assistant API connected |
| Blinking during OTA | Firmware update in progress |

### Physical Button (SW1)

| Press Duration | Action |
|----------------|--------|
| Short (50–500ms) | Toggle load ON/OFF |
| Long (3s+) | Restart the ESP |

### Safety Features

- **Configurable power-on behavior:** The "Power On Behavior" select entity in HA controls what happens after any power cycle or restart — three options are available directly from the device page in Home Assistant, no reflashing required:
  - **Always OFF** *(default, recommended for heaters and safety-critical loads)* — both relays are forced OFF at every boot via a RESET pulse, regardless of previous state
  - **Restore last state** — relays resume the state they were in before the power was cut, stored in device flash memory
  - **Always ON** — both relays are forced ON at every boot via a SET pulse

- **Dual-phase switching:** Both relays (L1 and L2) always switch simultaneously, safely interrupting both phases of the 240VAC circuit.

- **Bistable relay safety:** Because the relays are bistable (latching), they maintain their physical position even during a power failure. The boot RESET pulse ensures a known-good state is applied on every startup.

### Wiring Diagram (240VAC Single Load)

```
                    PTW04
                 ┌─────────┐
Power L1 ───────►│ Line L1 │──► Load L1
                 │  Relay  │         │
                 │         │       [Load]
                 │ Line L2 │         │
Power L2 ───────►│  Relay  │──► Load L2
                 └─────────┘
```

---

## Credits

- [LibreTiny](https://github.com/libretiny-eu/libretiny) — BK7231N support
- [ltchiptool](https://github.com/libretiny-eu/ltchiptool) — flash tool
- [bk7231tools](https://github.com/tuya-cloudcutter/bk7231tools) — firmware analysis
- [ESPHome](https://esphome.io) — firmware framework
- GPIO map derived from manual analysis of original firmware dump at offset `0x47690`

