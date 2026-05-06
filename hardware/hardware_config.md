# Hardware Configuration Reference — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research Reference Design

---

## 1. Philosophy: No Artificial Limits

This document specifies the hardware interfaces, power architecture, and configuration standards for SENTINEL-WEAR sensing nodes. It is a **reference design**, not a constraint specification.

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at their maximum defined speeds. Resolution, range, storage, and compute are not constrained by this document.
- **Modularity:** Each functional block (Sensing, Compute, Radio, Power) is defined by its interface, not a specific part number.
- **Connectivity Agnostic:** Supports BLE, UWB, custom ISM band, or any radio standard compatible with the MCU interface.
- **Storage Agnostic:** Designs support internal MCU flash, SD card, or external QSPI flash — user configures based on logging needs.

---

## 2. Core Node Architecture

Every SENTINEL-WEAR node shares a common architectural baseline.

```
┌────────────────────────────────────────────────────────────────┐
│                         Node Architecture                       │
├────────────────────────────────────────────────────────────────┤
│  MCU Core                                                       │
│  - ARM Cortex-M4F / M33 / RISC-V (configurable)                │
│  - FPU required for IMU fusion                                  │
│  - DSP extensions preferred for radar/audio processing          │
│  - Secure boot + crypto acceleration (recommended)              │
├────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Node Type)                   │
│  - mmWave Radar (SPI/UART)                                      │
│  - IMU (I2C/SPI)                                                │
│  - Acoustic (I2S/PDM)                                           │
│  - LiDAR/ToF (UART/SPI/I2C)                                     │
│  - Environmental (I2C)                                          │
│  - Event Sensor (MIPI/SPI parallel)                             │
│  - Identification Module (MIPI CSI / USB)                       │
├────────────────────────────────────────────────────────────────┤
│  Body-Area Network (BAN) Radio                                  │
│  - Interface: UART / SPI / SDIO                                 │
│  - Protocol: Software-defined (BLE 5.x / UWB / Proprietary)    │
│  - Antenna: PCB trace or ceramic chip antenna                   │
├────────────────────────────────────────────────────────────────┤
│  Power Management Unit (PMU)                                    │
│  - Battery Charger (linear or switching)                        │
│  - Fuel Gauge (I2C)                                             │
│  - Buck/Boost Regulators                                        │
│  - Power Path Management                                        │
├────────────────────────────────────────────────────────────────┤
│  User Interface (Optional per node)                             │
│  - Haptic Driver (GPIO + H-Bridge or I2C controller)            │
│  - Status LEDs (GPIO)                                           │
│  - Hardware Buttons (Reset, User, Kill Switch on ID nodes)      │
├────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                │
│  - SWD (Serial Wire Debug)                                      │
│  - UART Console                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Test Pad Standard (12-Pad BAN Interface)

All SENTINEL-WEAR nodes expose test pads for production jig pogo-pin contact.

| Pad | Signal | Description |
|---|---|---|
| 1 | `VCC_BATT` | Battery positive (3.7–4.2 V) |
| 2 | `GND` | Ground |
| 3 | `CHRG_IN+` | Charging input positive |
| 4 | `CHRG_IN-` | Charging input negative / GND reference |
| 5 | `BAN_RF_TEST` | BAN radio test point (direct antenna tap for power measurement) |
| 6 | `MCU_BOOT` | Boot mode select (HIGH = bootloader) |
| 7 | `MCU_RESET` | MCU reset (active low) |
| 8 | `KILL_SW_TEST` | Kill switch GPIO reading (identification sensor nodes only) |
| 9 | `HAPTIC_TEST` | Haptic actuator test point (AC coupled, for LRA resonance measurement) |
| 10 | `SWD_CLK` | SWD clock (for firmware flashing) |
| 11 | `SWD_DATA` | SWD data (for firmware flashing) |
| 12 | `UART_TX` | Debug UART TX (MCU to jig, 115200 baud) |

---

## 4. Power Architecture

### 4.1 Power Source

**Battery Chemistry:** Li-Po (Lithium Polymer) or Li-Ion cylindrical.
**Voltage:** 3.7V – 4.2V nominal.
**Capacity:** **Unconstrained.** Design supports 100 mAh (miniature nodes) to 7000 mAh+ (belt controller). PMU must support the full range of charge currents.

### 4.2 Power Rails

| Rail | Voltage | Usage |
|---|---|---|
| `VBAT` | 3.7–4.2V | Direct battery — radio PA, haptic driver |
| `VCC_3V3` | 3.3V | Main digital — MCU, sensors, radio |
| `VCC_1V8` | 1.8V | Low-power peripherals — IMU, environmental |
| `VCC_5V` | 5.0V | High-power sensors — some LiDAR/ToF modules |

### 4.3 Charging

- **Interface:** USB-C (USB 2.0 minimum) or Qi wireless charging coil.
- **Standard:** USB-C PD optional for faster charging.
- **Charging IC:** Must be I²C controllable for programmable charge current (adapts to different battery sizes).
- **Safety:** Hardware overcharge cutoff at 4.25 V. Overdischarge cutoff at 3.0 V. NTC thermistor on cell with cutoff > 60°C during charge.

---

## 5. Sensing Interfaces

### 5.1 mmWave Radar

**Protocol:** SPI (Mode 0/3) or UART.
**Speed:** Unconstrained. Supports SPI > 20 MHz or UART > 4 Mbps.

| Signal | Direction | Description |
|---|---|---|
| `RADAR_nRST` | Output | Hardware reset |
| `RADAR_nCS` | Output | SPI chip select |
| `RADAR_IRQ` | Input | Data ready interrupt |
| `SPI_MOSI/MISO/CLK` | Bidirectional | Shared SPI bus |

**Power:** Dedicated low-noise LDO recommended.

---

### 5.2 IMU

**Protocol:** I²C (≤ 1 MHz Fast Mode Plus) or SPI (≤ 10 MHz).
**Interrupt:** `IMU_DRDY` (Input) — wake MCU on new sample.
**Power:** Ultra-low-power rail (`VCC_1V8`) preferred.

---

### 5.3 Acoustic (Microphone Array)

**Protocol:** I²S or PDM.
**Channels:** 2–4 microphones per node.
**Clocking:** MCU provides Master Clock (MCLK) and Bit Clock (BCLK).

---

### 5.4 LiDAR / ToF

**Protocol:** UART (common for integrated modules) or SPI.
**Speed:** Configurable baud rate (115200 to 921600+).
**Trigger:** `LIDAR_TRIG` (Output) — external trigger for synchronization with other sensors.

---

### 5.5 Environmental Sensors

**Protocol:** I²C.
**Sensors:** Temperature, Humidity, Pressure, Gas (VOC).
**Integration:** Shared I²C bus with IMU (distinct addresses).

---

### 5.6 Event Sensor

**Protocol:** MIPI CSI-2 (for high bandwidth) or proprietary SPI-like parallel.
**Data Rate:** Very high — requires DMA support on MCU.
**Notes:** Requires high-speed PCB routing. Route differential pairs with 100 Ω impedance; length-match within 5 mil.

---

## 6. Identification Module Interface (Pendant Node Only)

### 6.1 Camera Sensor

**Protocol:** MIPI CSI-2 (2-lane or 4-lane) or parallel DVP.
**Resolution:** **Unconstrained.** Supports VGA to 4K+.

### 6.2 Hardware Kill Switch — CRITICAL

```
  Battery ──[ SW1 Physical Slider ]──┬──> V_CAM (camera power rail)
                                      │
                                      └──> ID_KILL_STATUS GPIO ──> MCU
                                           (reads LOW when switch DISABLED = power cut)
```

**Implementation:**
1. **Physical Switch SW1:** DPST mechanical slider accessible to user.
2. **Pole 1:** Physically disconnects `V_CAM` power rail.
3. **Pole 2:** Physically disconnects MIPI data lines via analog switch (or grounds `ID_KILL_STATUS` GPIO).

**Firmware Requirement:**
- MCU reads `ID_KILL_STATUS` GPIO as first action after GPIO init.
- If DISABLED: halt. Never initialize, communicate with, or attempt to power the Identification Module.
- No software override path. No "emergency" bypass. Halt is unconditional.

---

## 7. Haptic Actuator Interface

**Driver IC:** TI DRV2605L (I2C, waveform library, auto-resonance detection, back-EMF feedback).
**Interface:** I2C.
**Actuator:** LRA (Linear Resonant Actuator) — preferred for distinct directional patterns.
**Amplitude Control:** Via DRV2605L I2C registers.
**Feedback:** `HAPTIC_SENSE` (Input) — back-EMF for active resonance sensing (when supported).

---

## 8. Body-Area Network (BAN) Radio

**Protocol:** UART, SPI, or SDIO.
**Antenna:** PCB trace antenna on main board or U.FL connector for external antenna.

**Supported Standards:**
- BLE 5.x (Long Range, Coded PHY) — primary.
- UWB (Ultra-Wideband) — optional, for high-precision time-sync.
- Proprietary ISM band — research track.

---

## 9. Weight Targets (Per Node Class — Research Targets, Not Specifications)

These targets are for evaluating comfort. They will be refined based on user testing.

| Node | Target Weight (g) | Includes |
|---|---|---|
| Pendant (without ID sensor) | 20–40 | PCB, battery, enclosure |
| Pendant (with ID sensor) | 40–70 | PCB, battery, ID sensor, enclosure |
| Bracelet (single) | 30–50 | PCB, battery, haptic, enclosure |
| Belt node | 100–300 | Compute, battery bank, all radios, enclosure |
| Anklet (single) | 50–100 | PCB, battery, haptic, enclosure |
| Eyewear (clip-on) | 10–20 | PCB, battery, event camera |

---

## 10. Node-Specific Configuration Summary

| Node | Sensors | Actuators | Compute | Power |
|---|---|---|---|---|
| Pendant | mmWave, IMU, Acoustic, ID (opt-in) | Haptic | Mid MCU (Cortex-M4) | 150–300 mAh |
| Bracelet | mmWave, IMU | Haptic | Low MCU (Cortex-M4) | 200–400 mAh |
| Belt | mmWave (down), IMU, Environmental | — | High MCU or SoM | 2000–7000 mAh |
| Anklet | ToF, IMU | Haptic | Low MCU | 300–500 mAh |
| Eyewear | Event Sensor, IMU | — | Low MCU (Cortex-M33) | 50–80 mAh |

---

## 11. Battery Chemistry & Safety Requirements

All nodes use LiPo (lithium polymer) cells at 3.7 V nominal.

- All cells: UL 1642 or IEC 62133 certified.
- Overcharge protection: MOSFET cutoff at 4.25 V.
- Overdischarge protection: cutoff at 3.0 V.
- Thermal protection: NTC thermistor on cell, cutoff above 60°C during charge.
- Short-circuit protection: PTC fuse or dedicated protection IC (e.g., TI BQ29700).

---

## 12. Hypoallergenic Material Requirements

All skin-contact surfaces must comply with **EN 1811:2011** (nickel release limits: 0.5 μg/cm²/week for body-piercing articles; 0.2 μg/cm²/week for other direct prolonged skin contact items).

**Compliant materials:**
- Grade 5 titanium (Ti-6Al-4V) — preferred for metal hardware
- 316L surgical stainless steel — within threshold
- Medical-grade silicone (ISO 10993 biocompatible) — for band material
- Hard-coat anodized aluminum — for enclosures (coating must not chip)
- Polycarbonate or PETG — for plastic components

**Non-compliant:**
- Standard aluminum alloys (6061, 7075) — nickel content exceeds limit
- Brass, zinc die-cast without certified plating

---

## 13. Debug & Development Interface

**Connector:** 10-pin ARM Cortex Debug Connector (0.05" pitch) or Tag-Connect footprint.

| Signal | Description |
|---|---|
| `SWDIO`, `SWCLK` | Serial Wire Debug |
| `nRST` | MCU Reset |
| `SWO` | Serial Wire Output (printf over SWO) |
| `UART_TX`, `UART_RX` | Console |
| `VCC`, `GND` | Power reference |

**Bootloader:** USB DFU or ROM bootloader for firmware updates without debug probe.
