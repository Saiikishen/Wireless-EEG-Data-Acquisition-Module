# EEG-ACQ-v1.0 — Wireless EEG Data Acquisition Module

A compact, rechargeable, single-channel EEG acquisition board based on the ExG Biopill analog front-end, ADS1115 16-bit ADC, and ESP32 wireless SoC. Designed for wearable biopotential monitoring with real-time WiFi/BLE data streaming.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Bill of Materials](#bill-of-materials)
3. [Component Descriptions](#component-descriptions)
4. [Schematic Description](#schematic-description)
5. [Power Architecture](#power-architecture)
6. [Signal Chain](#signal-chain)
7. [I²C Bus Configuration](#i2c-bus-configuration)
8. [PCB Layout Notes](#pcb-layout-notes)
9. [Electrical Specifications](#electrical-specifications)
10. [Firmware Guidance](#firmware-guidance)
11. [Assembly Notes](#assembly-notes)
12. [Safety Considerations](#safety-considerations)

---

## System Overview

```
EEG Electrodes
     │ (differential analog)
     ▼
┌─────────────┐     analog      ┌─────────────┐     I²C      ┌─────────────┐
│  ExG Biopill│ ──────────────► │  ADS1115    │ ───────────► │   ESP32     │
│  (U1)       │  diff. pair     │  (U2)       │  0x48        │   (U3)      │
└─────────────┘                 └─────────────┘              └──────┬──────┘
                                                                     │
                                                              WiFi / BLE
                                                                     │
                                                             Remote host / app

Power chain:
LiPo 3.7V → TP4056 (BT1/U5) → AMS1117-3.3 (U4) → 3.3V rail → all ICs
USB-C (J3) → TP4056 charging input
```

**Board dimensions:** 65 mm × 55 mm  
**Supply voltage:** 3.7 V nominal (LiPo), 3.3 V regulated system rail  
**Communication:** WiFi 802.11 b/g/n, Bluetooth 4.2 / BLE 5.0  
**ADC resolution:** 16-bit, programmable gain amplifier  
**Sample rate:** Up to 860 SPS (ADS1115)

---

## Bill of Materials

| Ref | Component | Value / Part No. | Package | Qty |
|-----|-----------|-----------------|---------|-----|
| U1 | ExG Biopill | SparkFun DEV-11100 or compatible | Module / header | 1 |
| U2 | ADS1115 | Texas Instruments ADS1115IDGSR | SOIC-16 | 1 |
| U3 | ESP32 | Espressif ESP32-WROOM-32E | Castellated module | 1 |
| U4 | AMS1117-3.3 | AMS AMS1117-3.3 | SOT-223 | 1 |
| U5 | TP4056 | TP4056-42-SOT8-5 | SOP-8 | 1 |
| BT1 | LiPo cell | 3.7 V, 500–1000 mAh, JST-PH 2-pin | Bare cell | 1 |
| J1 | ExG header | 6-pin 2.54 mm male/female | THT | 1 |
| J2 | Battery connector | JST-PH 2-pin | SMD | 1 |
| J3 | USB-C port | USB Type-C receptacle, 2-pin power | SMD | 1 |
| C1 | Bypass capacitor | 100 nF, X7R, 10 V | 0402 | 1 |
| C2 | Bypass capacitor | 100 nF, X7R, 10 V | 0402 | 1 |
| C3 | Output capacitor | 10 µF, X5R, 10 V | 0805 | 1 |
| TP1 | Test point | 3.3 V rail | — | 1 |
| TP2 | Test point | GND | — | 1 |

> **Note:** All bypass capacitors (C1, C2) should be placed as close as possible to the VCC pins of their respective ICs. C3 is the output bulk capacitor for the AMS1117.

---

## Component Descriptions

### U1 — ExG Biopill (EEG Analog Front-End)

The ExG Biopill is a biopotential acquisition module designed for EEG, ECG, and EMG signals. It incorporates:

- High common-mode rejection ratio (CMRR) instrumentation amplifier
- Bandpass filtering tuned to the EEG frequency range (0.5–100 Hz typical)
- Right-leg drive (RLD) circuit for common-mode noise suppression
- Differential output pair (OUT+, OUT−) connected to the ADC

The module is interfaced via a 6-pin header (J1) carrying: VCC, GND, IN+, IN−, SDA, SCL. The SDA/SCL lines are reserved for future configuration; in this design the differential analog outputs are the primary signal path.

### U2 — ADS1115 (16-bit ADC)

A precision, low-power, 16-bit sigma-delta ADC from Texas Instruments featuring:

- Four single-ended or two differential input channels
- Programmable gain amplifier (PGA): ±0.256 V to ±6.144 V full-scale
- I²C interface, address configurable via ADDR pin
- Internal oscillator, no external clock required
- Operating supply: 2.0–5.5 V

In this design, AIN0+ and AIN0− receive the differential analog output from the ExG Biopill. The recommended PGA setting for EEG is **±0.512 V or ±1.024 V** depending on electrode placement and gain stage output level.

### U3 — ESP32-WROOM-32 (Wireless MCU)

Espressif's dual-core Xtensa LX6 module operating at up to 240 MHz, featuring:

- Integrated 802.11 b/g/n WiFi and Bluetooth 4.2 / BLE 5.0
- 4 MB flash (WROOM-32E variant: also includes PSRAM option)
- Hardware I²C peripheral on GPIO21 (SDA) and GPIO22 (SCL)
- 3.3 V operating voltage, peak current ~300 mA during WiFi TX bursts
- PCB trace antenna with keep-out zone (no copper, no vias within 3 mm)

The ESP32 acts as the I²C master, polling the ADS1115 at the configured sample rate and transmitting data packets over WiFi UDP or BLE notifications to a host application.

### U4 — AMS1117-3.3 (LDO Voltage Regulator)

A low-dropout linear regulator providing a stable 3.3 V supply rail from the LiPo battery output.

- Input voltage range: 4.75–15 V (min dropout ~1.1 V at full load)
- Output: 3.3 V ± 1%, 800 mA maximum
- Dropout voltage: ~1.1 V at 800 mA; typically ~0.6 V at 100 mA
- Continues regulating until LiPo battery drops to approximately **3.4–3.5 V**

The AMS1117 is essential because the LiPo cell voltage ranges from 3.0 V (depleted) to 4.2 V (fully charged). Without regulation, this swing would introduce noise into the ADC reference and supply rails, corrupting EEG amplitude accuracy.

### U5 — TP4056 (LiPo Charger IC)

A single-cell lithium-ion battery charger with integrated protection logic.

- Charges via USB-C input (J3), 5 V input
- Programmable charge current via external resistor (default ~1 A; reduce for small cells)
- End-of-charge voltage: 4.2 V
- Includes overcharge, over-discharge, and short-circuit protection in full-module versions
- Status indicators: CHRG (charging), STDBY (charge complete)

> **Important:** If using a bare TP4056 IC (not a pre-built module), add a 2kΩ programming resistor on the PROG pin to set charge current to ~500 mA, appropriate for cells between 500–1000 mAh. Charging at 1C or lower is recommended.

---

## Schematic Description

### Nets

| Net name | Description |
|----------|-------------|
| `+3V3` | Regulated 3.3 V system rail, sourced from AMS1117 output |
| `VBAT` | Raw LiPo output, 3.0–4.2 V, feeds AMS1117 input |
| `GND` | Common ground plane |
| `SDA` | I²C data line, GPIO21 on ESP32 |
| `SCL` | I²C clock line, GPIO22 on ESP32 |
| `AIN0P` | ADS1115 positive differential input from ExG Biopill OUT+ |
| `AIN0N` | ADS1115 negative differential input from ExG Biopill OUT− |
| `VUSB` | USB-C 5 V input to TP4056 |

### Net topology

The `+3V3` rail runs as a horizontal bus at the top of the schematic, with decoupling capacitors (C1, C2) placed at the VCC pins of U2 and U3 respectively. The `GND` rail runs as a horizontal bus at the bottom. All IC ground connections drop vertically to this rail.

The differential analog signal path from U1 to U2 is kept short and routed as a matched-length pair. The I²C bus connects U2 to U3 directly, with no additional pull-up resistors required if the ESP32's internal pull-ups are enabled in firmware (GPIO_PULLUP_ENABLE), or optionally add 4.7 kΩ pull-ups to +3V3 for greater noise immunity.

The ADDR pin of U2 is tied directly to GND, setting the I²C device address to **0x48**.

---

## Power Architecture

```
USB-C 5V (J3)
      │
      ▼
  TP4056 (U5)
  Charge controller
      │ VBAT (3.0–4.2V)
      ▼
 LiPo Cell (BT1)
      │ VBAT
      ▼
  AMS1117-3.3 (U4)
  LDO Regulator
      │ +3V3 (stable 3.3V)
      ├──► ExG Biopill (U1)
      ├──► ADS1115 (U2)  + C1 100nF local bypass
      └──► ESP32 (U3)    + C2 100nF local bypass
                           C3 10µF output bulk cap
```

### Power budget (estimated)

| Component | Typical current | Peak current |
|-----------|----------------|--------------|
| ExG Biopill (U1) | ~2 mA | ~5 mA |
| ADS1115 (U2) | ~0.15 mA | ~0.3 mA |
| ESP32 (U3) — idle | ~20 mA | — |
| ESP32 (U3) — WiFi TX | ~80 mA | ~300 mA |
| AMS1117 quiescent | ~5 mA | — |
| **Total (WiFi active)** | **~107 mA** | **~310 mA** |

A 500 mAh LiPo cell provides an estimated **~4–5 hours** of continuous operation with WiFi active. Using BLE instead of WiFi reduces ESP32 current to ~20–30 mA, extending battery life to approximately **12–15 hours**.

---

## Signal Chain

```
Scalp electrodes
       │  µV-level biopotential
       ▼
ExG Biopill (U1)
  - Instrumentation amp (gain ~1000×)
  - Bandpass filter: 0.5 Hz – 100 Hz
  - CMRR > 80 dB
  - Output: differential, 0–3.3V swing
       │  mV-level differential analog
       ▼
ADS1115 (U2)
  - AIN0+ / AIN0− differential input
  - PGA: ±1.024V recommended (gain = 2)
  - 16-bit Σ-Δ conversion
  - Sample rate: up to 860 SPS
  - Output: 16-bit two's complement via I²C
       │  I²C (SDA/SCL)
       ▼
ESP32 (U3)
  - Reads ADS1115 conversion register (0x00)
  - Buffers samples in ring buffer
  - Transmits via WiFi UDP / BLE notification
       │  Wireless
       ▼
Host application (PC / mobile)
```

### Recommended ADS1115 register configuration

| Register | Field | Value | Description |
|----------|-------|-------|-------------|
| Config (0x01) | MUX | 000 | AIN0+ / AIN0− differential |
| Config (0x01) | PGA | 010 | ±1.024 V full scale |
| Config (0x01) | MODE | 0 | Continuous conversion |
| Config (0x01) | DR | 111 | 860 SPS |
| Config (0x01) | COMP_QUE | 11 | Disable comparator |

---

## I²C Bus Configuration

| Parameter | Value |
|-----------|-------|
| Bus speed | 400 kHz (Fast mode) |
| ADS1115 address | 0x48 (ADDR → GND) |
| ESP32 SDA pin | GPIO21 |
| ESP32 SCL pin | GPIO22 |
| Pull-up resistors | 4.7 kΩ to +3V3 (or enable internal ESP32 pull-ups) |

The ADDR pin determines the 7-bit I²C address of the ADS1115 as follows:

| ADDR connection | Address |
|----------------|---------|
| GND | 0x48 ← **used in this design** |
| VCC | 0x49 |
| SDA | 0x4A |
| SCL | 0x4B |

To add a second ADS1115 (for additional channels), connect its ADDR pin to VCC and address it at 0x49. Both devices share the same SDA/SCL lines.

---

## PCB Layout Notes

### Layer stack (recommended 2-layer)

| Layer | Purpose |
|-------|---------|
| Top copper | Signal traces, component pads, power traces |
| Bottom copper | Continuous GND plane |

### Trace widths

| Net | Width |
|-----|-------|
| +3V3 power rail | 0.5 mm |
| GND | Pour / flood fill |
| I²C (SDA, SCL) | 0.2 mm |
| Analog differential pair (AIN0+/−) | 0.15 mm, length-matched |
| USB-C / VBAT power | 0.5–1.0 mm |

### Critical placement rules

1. **Analog front-end isolation:** Place U1 (ExG Biopill) and U2 (ADS1115) on the opposite side of the board from U3 (ESP32). The ESP32's switching regulator and RF emissions are significant noise sources for a µV-level analog front-end.

2. **Antenna keep-out zone:** Maintain a minimum 3 mm copper-free keep-out zone around the ESP32's PCB trace antenna. No copper pours, vias, or traces in this region on any layer.

3. **Bypass capacitors:** C1 and C2 must be placed within 1 mm of the VCC pin of their respective ICs, with the ground via directly adjacent.

4. **Differential pair routing:** Route AIN0+ and AIN0− as a tightly coupled differential pair (gap ≤ 0.15 mm), with equal length. Avoid running them parallel to I²C or power traces for more than 5 mm.

5. **GND plane continuity:** Do not cut the bottom copper GND plane under the analog signal path. A solid plane is critical for shielding and a stable voltage reference for the ADC.

6. **USB-C connector placement:** J3 must be accessible from the board edge. Ensure the pad footprint matches your selected USB-C receptacle's datasheet exactly — USB-C pad geometries vary between manufacturers.

---

## Electrical Specifications

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Supply voltage (VBAT) | 3.0 | 3.7 | 4.2 | V |
| Regulated supply (+3V3) | 3.267 | 3.3 | 3.333 | V |
| System current (WiFi active) | — | 107 | 310 | mA |
| System current (BLE active) | — | 30 | 80 | mA |
| ADC resolution | — | 16 | — | bit |
| ADC sample rate | — | 860 | 860 | SPS |
| ADC input range (PGA ±1.024V) | −1.024 | — | +1.024 | V |
| ADC LSB size (PGA ±1.024V) | — | 31.25 | — | µV |
| EEG bandwidth (ExG Biopill) | 0.5 | — | 100 | Hz |
| Operating temperature | −20 | 25 | 70 | °C |
| Board dimensions | — | 65 × 55 | — | mm |

---

## Firmware Guidance

The following pseudo-code outlines the core firmware loop. Implement using the Arduino framework or ESP-IDF.

```cpp
#include <Wire.h>
#include <Adafruit_ADS1X15.h>
#include <WiFiUdp.h>

Adafruit_ADS1115 ads;
WiFiUDP udp;

void setup() {
  Wire.begin(21, 22);          // SDA=GPIO21, SCL=GPIO22
  ads.setGain(GAIN_TWO);       // ±1.024V full scale
  ads.setDataRate(RATE_ADS1115_860SPS);
  ads.begin(0x48);             // ADDR → GND

  WiFi.begin(SSID, PASSWORD);
  udp.begin(UDP_PORT);
}

void loop() {
  int16_t raw = ads.readADC_Differential_0_1();
  float voltage_uV = raw * 31.25f;  // µV per LSB at ±1.024V

  // Pack sample + timestamp into UDP payload
  uint8_t buf[6];
  uint32_t ts = micros();
  memcpy(buf, &ts, 4);
  memcpy(buf + 4, &raw, 2);
  udp.beginPacket(HOST_IP, HOST_PORT);
  udp.write(buf, 6);
  udp.endPacket();
}
```

> For continuous acquisition without polling delay, configure the ADS1115 in **continuous conversion mode** and use the ALRT/RDY pin (if broken out) as a data-ready interrupt on a spare ESP32 GPIO rather than polling via I²C.

---

## Assembly Notes

1. Solder SMD components (U2, U4, U5, C1–C3) before through-hole and connectors.
2. Solder the ESP32-WROOM-32 module using castellated pads; ensure full solder contact on all pads, especially the ground pad row.
3. Do not apply flux or solder to the antenna area of the ESP32 module.
4. Connect the LiPo cell last, after all solder work is complete, to avoid shorts during assembly.
5. Verify +3V3 rail voltage at TP1 before connecting U1 or any external electrodes.
6. Measure quiescent current at TP1/TP2 before powering the full system; expected idle draw is ~25–30 mA.

---

## Safety Considerations

> ⚠️ **Body-contact device warning:** This circuit is designed to interface directly with human skin via EEG electrodes. The following precautions are mandatory.

- **Isolation:** This design is **not medically isolated**. It must not be used in clinical or diagnostic settings without a certified isolation barrier between the patient and any mains-connected equipment.
- **Battery protection:** Ensure the LiPo cell includes or is paired with a protection circuit module (PCM) to prevent over-discharge, over-charge, and short-circuit conditions.
- **Electrode gel:** Use only conductive gel certified for skin contact. Avoid broken skin.
- **Current limitation:** The ExG Biopill's instrumentation amplifier input impedance and the ADC input protection diodes limit injected current to well below IEC 60601-1 patient auxiliary current limits under normal operation, but this has not been formally verified for this design.
- **USB charging:** Do not wear the device while simultaneously charging via USB-C unless an isolation transformer is used on the USB supply.

---

*EEG-ACQ-v1.0 · Designed with ExG Biopill + ADS1115 + ESP32 · Released for reference use*
