# Hardware Configuration Reference — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research Reference Design

---

## 1. Philosophy: No Artificial Limits

This document specifies the hardware interfaces, power architecture, and configuration standards for SENTINEL-WEAR sensing nodes. It is a **reference design**, not a constraint specification.

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at maximum speeds. Resolution, storage, compute are configuration choices.
- **Modularity:** Each functional block defined by interface, not part number.
- **Connectivity Agnostic:** BLE 5.x primary; UWB optional; Wi-Fi for app connectivity.
- **Storage Agnostic:** microSD, internal flash, or network — user chooses.
- **User-Controlled Data Handling:** No system-imposed restrictions on storage, streaming, or retention.

---

## 2. Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      SENTINEL-WEAR Node                           │
├──────────────────────────────────────────────────────────────────┤
│  MCU Core                                                         │
│  - ARM Cortex-M4F / M33 / M7 / Linux SoM (configurable)          │
│  - FPU required for IMU fusion                                    │
│  - DSP extensions preferred for radar and audio processing        │
├──────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Node Type)                     │
│  - mmWave Radar (SPI/UART)                                        │
│  - IMU (I2C/SPI)                                                  │
│  - Acoustic / Microphone Array (I2S/PDM)                          │
│  - LiDAR/ToF (UART/SPI/I2C)                                       │
│  - Environmental (I2C)                                            │
│  - Event Sensor (MIPI/SPI parallel)                               │
│  - Camera (MIPI CSI-2 / DVP) — any variant, any node             │
│  - 360° Camera Array (multiple MIPI lanes + HW sync)              │
├──────────────────────────────────────────────────────────────────┤
│  Storage Interface (Optional Per Configuration)                   │
│  - microSD card (SPI, up to 2 TB) — raw recordings, SLAM maps    │
│  - Internal QSPI NOR Flash — metadata, logs, model weights        │
├──────────────────────────────────────────────────────────────────┤
│  Body-Area Network (BAN) Radio                                    │
│  - BLE 5.x (mandatory, integrated or module)                      │
│  - UWB optional (Qorvo DW3000 class) for precision time sync      │
├──────────────────────────────────────────────────────────────────┤
│  Power Management                                                 │
│  - Battery Charger (linear or switching)                          │
│  - Fuel Gauge (I2C)                                               │
│  - Buck/Boost Regulators                                          │
│  - Optional Belt Power Input (high-current from belt battery)     │
├──────────────────────────────────────────────────────────────────┤
│  User Interface (Optional Per Node)                               │
│  - Haptic Driver (I2C DRV2605L) + LRA or ERM actuator            │
│  - Status LEDs (GPIO)                                             │
│  - Hardware Power Switch (optional, user-installed) for cameras   │
├──────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                  │
│  - SWD (Serial Wire Debug)                                        │
│  - UART Console (115200 baud)                                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Test Pad Standard (12-Pad BAN Interface)

| Pad | Signal | Description |
|---|---|---|
| 1 | `VCC_BATT` | Battery positive (3.7–4.2 V) |
| 2 | `GND` | Ground |
| 3 | `CHRG_IN+` | Charging input positive |
| 4 | `CHRG_IN-` | Charging input negative |
| 5 | `BAN_RF_TEST` | BAN radio test point (antenna tap) |
| 6 | `MCU_BOOT` | Boot mode select (HIGH = bootloader) |
| 7 | `MCU_RESET` | MCU reset (active low) |
| 8 | `CAM_HW_SW_TEST` | Camera hardware switch GPIO reading (if switch installed on this unit) |
| 9 | `HAPTIC_TEST` | Haptic actuator test point (AC coupled, LRA resonance) |
| 10 | `SWD_CLK` | SWD clock |
| 11 | `SWD_DATA` | SWD data |
| 12 | `UART_TX` | Debug UART TX |

---

## 4. 360° Curved Pendant Interface Standard

The 360° curved pendant requires additional interface definitions beyond the standard node header.

### 4.1 Camera Array Interface (Per Camera Module Position)

| Signal | Description | Routing Notes |
|---|---|---|
| `CAM_n_MIPI_CLK+` | MIPI CLK+ for camera n | 100 Ω differential, length-match within 2 mil |
| `CAM_n_MIPI_CLK-` | MIPI CLK- for camera n | Pair with above |
| `CAM_n_MIPI_D0+` | MIPI DATA0+ for camera n | 100 Ω differential |
| `CAM_n_MIPI_D0-` | MIPI DATA0- for camera n | |
| `CAM_FSYNC` | Hardware sync pulse broadcast to all cameras | Single GPIO from MCU to all camera FSYNC pins |
| `CAM_PWDN_n` | Power-down control per camera (optional) | GPIO per camera for selective shutdown |

### 4.2 Vision MCU to Belt Node High-Speed Link (Variant B processing)

| Signal | Description |
|---|---|
| `USB3_TX+`, `USB3_TX-` | USB 3.x SuperSpeed TX differential pair |
| `USB3_RX+`, `USB3_RX-` | USB 3.x SuperSpeed RX differential pair |
| `USB3_GND` | USB 3.x ground reference |

### 4.3 Belt Power Input (Extended Runtime)

| Signal | Description |
|---|---|
| `V_BELT_IN` | 5V input from belt node battery bank (via necklace cable conductor) |
| `V_BELT_GND` | Return ground |
| `V_BELT_EN` | Power enable signal (belt controller controls when to supply power) |

---

## 5. Power Architecture

### 5.1 Standard Nodes

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `VBAT` | 3.7–4.2V | LiPo battery | Direct to radio PA, haptic driver peak |
| `VCC_3V3` | 3.3V | LDO from VBAT | MCU and all digital sensors |
| `VCC_1V8` | 1.8V | LDO | Low-power IMU, environmental |
| `V_CAM` | 3.3V or 2.8V | Regulated | Camera module (software or optional HW switch gated) |

### 5.2 360° Pendant Additional Rails

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_CAMS` | 2.8V / 3.3V | Regulated, ganged | All N camera modules |
| `V_ISP` | 1.8V / 1.0V | Vision MCU supply | Vision MCU core / IO |
| `V_EXT` | 5V | From belt node (optional) | Belt-supplied power for extended runtime |

### 5.3 Node Weight and Battery Targets

| Node | Target Weight (g) | Battery (mAh) |
|---|---|---|
| Pendant — Standard | 30–60 | 150–300 |
| Pendant — 360° Curved | 50–120 | 400–1000 |
| Pendant — Medallion | 80–150 | 800–1500 |
| Bracelet — Minimal | 25–40 | 150–250 |
| Bracelet — Standard | 30–50 | 200–400 |
| Bracelet — Extended (Camera) | 40–60 | 300–500 |
| Belt (all variants) | 100–300 | 2000–7000 |
| Anklet — Minimal | 40–70 | 300 |
| Anklet — Extended | 50–100 | 400–500 |
| Eyewear — Clip-on | 5–15 | 50–80 |
| Eyewear — Frame / Headband | 10–30 | 80–150 |

*All values are research targets pending test measurement.*

---

## 6. Camera Privacy Controls — Available Options

All camera privacy controls are user-selected. None are mandatory by the architecture.

**Option A — Hardware Power Switch (user-installed):**
- Physical switch cuts `V_CAM` or `V_CAMS` power rail.
- When DISABLED: camera is physically unpowered. No software override.
- PCB provides footprint for this switch on all camera-capable nodes.
- MOSFET on the `V_CAM` rail; switch controls gate drive.
- Firmware reads `CAM_HW_SW_TEST` GPIO at boot if pin is populated.
- Users who want unconditional physical camera disable select this option.

**Option B — Software-Only Control:**
- MCU controls `V_CAM` via MOSFET GPIO without any physical switch.
- Software scheduling, detection-triggered activation, or manual control.
- Simpler hardware (no switch component).
- Users who prefer flexibility over guaranteed physical disable select this option.

**Option C — Always-On Camera:**
- Camera powered continuously; no gating.
- Data handling (storage, streaming, retention) configured in `sentinel-wear.toml`.
- Useful for continuous recording deployments.

---

## 7. Sensing Interfaces

### 7.1 mmWave Radar

| Signal | Direction | Description |
|---|---|---|
| `RADAR_nRST` | Output | Hardware reset |
| `RADAR_nCS` | Output | SPI chip select |
| `RADAR_IRQ` | Input | Data ready interrupt |
| `SPI_MOSI/MISO/CLK` | Bidirectional | Shared SPI bus |

### 7.2 IMU

- I²C (≤ 1 MHz Fast Mode Plus) or SPI (≤ 10 MHz)
- `IMU_DRDY` interrupt for new-sample wake

### 7.3 Acoustic (Microphone Array)

- I²S or PDM (MCU provides MCLK and BCLK)
- 3–6 microphones supported (topology per node variant)

### 7.4 LiDAR / ToF

- UART (common for integrated modules) or SPI
- `LIDAR_TRIG` for external synchronization with other sensors

### 7.5 Environmental

- I²C, shared bus with IMU

### 7.6 Event Sensor

- MIPI CSI-2 or proprietary SPI-like parallel
- DMA required for high-bandwidth event rates

### 7.7 Camera

- MIPI CSI-2 (standard single camera or 360° array)
- 100 Ω differential impedance; length-match within 2 mil per pair
- DVP (8-bit parallel) for lower-resolution lower-bandwidth modules

---

## 8. Body-Area Network Radio

**Primary:** BLE 5.x integrated MCU radio.
**Optional UWB:** Qorvo DW3000 class (sub-ns timestamping). Particularly valuable for:
- Precision gait phase synchronization between anklet pairs
- 360° pendant camera sync reference
- High-accuracy body-frame reconstruction

---

## 9. Battery Chemistry & Safety

All nodes use LiPo cells at 3.7V nominal:
- UL 1642 or IEC 62133 certified cells required
- Overcharge protection: MOSFET cutoff at 4.25V
- Overdischarge protection: cutoff at 3.0V
- Thermal protection: NTC thermistor, cutoff above 60°C during charge
- Short-circuit protection: PTC fuse or dedicated protection IC

---

## 10. Hypoallergenic Material Requirements

EN 1811:2011 nickel release compliance for all skin-contact surfaces:
- Grade 5 titanium (Ti-6Al-4V) — preferred metal
- 316L surgical stainless steel — alternative
- Medical-grade silicone (ISO 10993) — band material
- Hard-coat anodized aluminum — enclosures (coating must not chip)
- Polycarbonate or PETG — plastic components

Non-compliant (do not use for skin-contact): 6061/7075 aluminum alloys, brass, unplated zinc die-cast.

---

## 11. Debug Interface

10-pin ARM Cortex Debug Connector (0.05" pitch) or Tag-Connect footprint:
- `SWDIO`, `SWCLK` — Serial Wire Debug
- `nRST` — MCU Reset
- `SWO` — Serial Wire Output (printf over SWO)
- `UART_TX`, `UART_RX` — Console at 115200 baud
- Bootloader: USB DFU or ROM bootloader for OTA firmware updates
