# Pendant Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Neck/chest — worn as a necklace or pendant
**Primary Role:** Primary Sensing & Identification Node (Upper Hemisphere Coverage)
**Form Factor:** Pendant / Chest-Mounted
**Status:** Reference Design v1.0
**License:** CERN-OHL-S v2

---

## 1. Overview

The Pendant Node is the highest-information-value wearable node. Its chest/neck position gives it 360° azimuthal view of the environment at chest height — the most relevant sensing plane for approaching humans and objects. It also hosts:
- The primary acoustic sensor for sound-event direction and material classification.
- The optional identification sensor (visual/biometric — opt-in, kill-switch gated).
- IMU for body-frame pose contribution and orientation reference.

---

## 2. Design Philosophy

### 2.1 Limitless Configuration

This is a **reference design**, not a constraint. The architecture supports substitution of components to match research needs or available supply.

- **Sensors:** The reference design uses a specified set, but the PCB layout accommodates alternative footprints where possible.
- **Processing:** The MCU selection provides headroom for algorithm development.
- **Connectivity:** No hardcoded connectivity assumptions. Supports local-only logging, BLE streaming, and Wi-Fi backhaul.

### 2.2 Sensing vs. Identification Layer Separation

The Pendant Node physically separates the **Sensing Layer** from the **Identification Layer**:
- **Sensing Layer:** Always-on, privacy-preserving (mmWave, IMU, Acoustic). Powered by the main power rail.
- **Identification Layer:** The camera/biometric module. **Controlled by a dedicated hardware kill switch (SW1).** The MCU firmware must read the kill-switch state at boot and halt execution if the switch is in the "disabled" position. No software override is permitted.

---

## 3. High-Level Block Diagram

```
[ Power Input / Charger ]
          │
    [ PMIC / Power Mgmt ]───────────────────┐
          │                                 │
          ▼                                 ▼
[ Main MCU (nRF5340 / STM32H7) ]    [ Haptic Driver + LRA ]
          │
          ├───[ mmWave Radar (SPI/UART) ]
          │
          ├───[ IMU (SPI) ]
          │
          ├───[ Microphone Array (I2S/PDM) ]
          │
          ├───[ Environmental Sensors (I2C) ]
          │
          ├───[ Event Sensor (MIPI/SPI) ] (Optional)
          │
          └───[ Identification Module ]
                       │
                       ├───[ Camera Module (MIPI CSI / SPI) ]
                       │
                       └───[ Kill Switch SW1 (Physical DPST) ]
```

---

## 4. Candidate Component Set (Test Variants — Not Locked)

All of the following are test candidates. Architecture explicitly supports testing multiple variants at each position. Substitutions are expected and encouraged.

### 4.1 Microcontroller (MCU)

**Reference Part:** Nordic Semiconductor nRF5340
- **Architecture:** Dual-core Arm Cortex-M33 (Application Core 128 MHz + Network Core 64 MHz)
- **Flash:** 1 MB Application Flash, 256 KB RAM
- **Connectivity:** Bluetooth 5.1 / LE Audio, IEEE 802.15.4 (Thread), SPI, I2C, I2S, PDM, UART
- **Reasoning:** Integrated BLE radio (critical for BAN), high-performance core for sensor fusion, ample memory for logging and algorithm development.

**Alternative:** STM32WB55 (Cortex-M4 + M0+, BLE 5.0, ultra-low power, 2.4 GHz)

### 4.2 Sensors

#### mmWave Radar
- **Variant A:** Texas Instruments IWR6843ISK-ODS (60–64 GHz, on-chip antenna, FMCW, DSP/ARM integrated) — evaluation module for early testing
- **Variant B:** Acconeer XR112 module (60 GHz, SPI, ultra-compact — preferred for wearable integration)
- **Variant C:** Infineon BGT60ATR24C (60 GHz, MMIC, requires external antenna design)

**Interface:** SPI + GPIOs (Data Ready, Reset). Power: 3.3 V.
**Coverage:** Antenna oriented to maximize forward and lateral hemisphere coverage from chest position.

#### IMU
- **Variant A:** Bosch BMI270 (6-axis, OIS, ultra-low power, SPI/I2C, 0.1 mg noise density) — preferred for noise floor
- **Variant B:** TDK InvenSense ICM-42688-P (6-axis, 0.07 mg noise density, SPI, FIFO, on-chip FIFO)
- **Variant C:** ST LSM6DSV (6-axis + temperature, iNEMO 7-axis, built-in step/gesture, SPI)

**Selection criteria:** Noise density and bias stability are primary. IMU is critical for body-frame drift correction.

#### Microphone Array (Acoustic)
- **Variant A:** Knowles SPH0645LM4H × 3 (I2S PDM MEMS, compact, 65 dBA SNR) — 3-element planar array
- **Variant B:** ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR) — 4-element tetrahedral on flexible PCB
- **Variant C:** InvenSense ICS-40180 × 4 (analog MEMS, 65 dBA SNR)

**Array geometry:** Planar 3-element for space-constrained pendants; tetrahedral 4-element for better 3D direction-of-arrival in larger variants.

#### Environmental Sensors (Optional)
- **Variant A:** Bosch BME680 (Temperature, Humidity, Pressure, Gas/VOC — single chip, I2C/SPI)
- **Variant B:** Bosch BME688 (as BME680 + AI gas sensing)

#### Event Sensor (Optional / Advanced Research)
- **Variant A:** Prophesee Metavision EVK3 (640×480, USB, lower power)
- **Variant B:** Sony IMX637 sensor on custom carrier — research variant

Routes high-speed differential pairs to a connector for external module. Base design includes pads for these traces.

### 4.3 Identification Module (Opt-In Layer — All Variants Under Research)

None mandated. **Kill switch is mandatory before any variant can be activated.**

**Visual (Camera-Based):**
- **Variant A:** OmniVision OV2640 (2 MP, small package, MIPI/DVP) — camera-based face detection
- **Variant B:** Arducam IMX219 (8 MP, CSI, configurable wide or narrow angle)
- **Variant C:** Luxonis OAK-D Lite (stereo + neural inference on-module) — self-contained identification

**Ultra-Low Power:**
- **Variant D:** HiMax HM01B0 (QVGA, 2.1 mW, ultra-low power) — minimal footprint, first-stage detection on-chip

**Non-Visual:**
- **Variant E:** None — pendant in audio-only identification mode via voice biometrics

**Interface:** MIPI CSI-2 (direct to MCU) or SPI for lower-res modules.

### 4.4 Power Management

**Reference Part:** Nordic nPM1300 PMIC
- Battery charging (LiPo/Li-Ion), Buck regulators, Fuel Gauge.
- **Battery:** LiPo 3.7V, 150–300 mAh (jewelry weight budget: 10–15 g battery maximum).
- **Variants under test:** 150 mAh (ultra-thin, 1.5 mm), 200 mAh (standard pendant), 300 mAh (bulkier, 5 mm).
- **Charging:** USB-C or magnetic POGO pin.

### 4.5 Haptic

**Reference Part:** TI DRV2605L Haptic Driver (I2C, waveform library, auto-resonance detection).
**Actuator:** LRA (Linear Resonant Actuator), 10 mm diameter.

### 4.6 BAN Radio

- **Variant A:** BLE 5.3 via nRF5340 network core (if MCU is nRF5340)
- **Variant B:** UWB via Qorvo DW3000 (sub-ns timestamping, <1 m ranging accuracy, higher power — research variant)

---

## 5. Pin Mapping & Configuration

*(Reference mapping for nRF5340. Adjust for your specific MCU selection and pin availability.)*

### 5.1 mmWave Radar

| Signal | nRF5340 Pin | Description |
|---|---|---|
| `SPI_MOSI` | P1.00 | SPI Master Out Slave In |
| `SPI_MISO` | P1.01 | SPI Master In Slave Out |
| `SPI_CLK` | P1.02 | SPI Clock |
| `RADAR_CS` | P1.03 | Chip Select |
| `RADAR_RST` | P1.04 | Reset |
| `RADAR_IRQ` | P1.05 | Data Ready Interrupt |

### 5.2 IMU

| Signal | nRF5340 Pin | Description |
|---|---|---|
| `IMU_CS` | P1.06 | Chip Select |
| `IMU_INT` | P1.07 | Data Ready Interrupt |
| `SPI_MOSI` | P1.00 | (Shared with Radar — different CS) |
| `SPI_MISO` | P1.01 | (Shared) |
| `SPI_CLK` | P1.02 | (Shared) |

### 5.3 Acoustic (PDM Microphones)

| Signal | nRF5340 Pin | Description |
|---|---|---|
| `PDM_CLK` | P1.08 | Clock Output |
| `PDM_DATA` | P1.09 | Data Input |

### 5.4 Identification Module (Camera)

| Signal | nRF5340 Pin | Description |
|---|---|---|
| `CAM_PWDN` | P1.10 | Power Down Control |
| `KILL_SW` | P1.11 | **Hardware Kill Switch Input (Read at boot — no bypass)** |
| `MIPI_CSI_CLK+` | Dedicated | High-speed differential pair |
| `MIPI_CSI_D0+` | Dedicated | High-speed differential pair |

### 5.5 Power, Haptics, & Debug

| Signal | nRF5340 Pin | Description |
|---|---|---|
| `HAPTIC_I2C_SDA` | P1.13 | I2C Data (DRV2605L) |
| `HAPTIC_I2C_SCL` | P1.14 | I2C Clock |
| `ENV_I2C_SDA` | P1.13 | I2C Data (shared bus, diff address) |
| `SWD_CLK` | P0.18 | Debug |
| `SWD_DIO` | P0.20 | Debug |
| `USB_D+` | Dedicated | USB Data+ |
| `USB_D-` | Dedicated | USB Data- |
| `BAT_SENSE` | AIN0 | Battery Voltage (via resistor divider) |

---

## 6. Power Architecture

### 6.1 Power Rails

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7V – 4.2V | Direct battery | Radio PA, haptic driver |
| `V_SYS` | 3.3V | nPM1300 regulated | MCU, Sensors, Radio |
| `V_MCU` | 3.3V | Regulated | Always-on |
| `V_RADAR` | 3.3V | Always-on rail | Primary sensing |
| `V_CAM` | 3.3V or 2.8V | **Switched via MOSFET + Kill Switch** | Identification module only |

### 6.2 Kill Switch Logic — CRITICAL

The Kill Switch (`SW1`) is a physical DPST (Double Pole Single Throw) slider switch.

- **Pole 1:** Breaks the `V_CAM` power rail to the identification module.
- **Pole 2:** Breaks the MIPI data lines (analog switch in series) OR connects a `KILL_SW` GPIO signal high/low.

**Firmware Requirement (no override):**
- The MCU **must** read the `KILL_SW` GPIO pin as the first action after GPIO init.
- If `KILL_SW` is LOW (switch DISABLED), the MCU **must halt** and **never attempt to power on, initialize, or communicate with the Identification Module.**
- The only permitted activity in this state: respond to BAN health queries with `KillSwitchEngaged` status.
- The slider position must be visible to the wearer without removing the pendant.

**Circuit note:** A second GPIO path reads MOSFET drain to confirm `V_CAM` is truly zero even if the MCU GPIO misreads. Hardware confirmation is belt-and-suspenders.

---

## 7. Form Factor Constraints

- **Dimensions:** Maximum 50 mm × 50 mm (excluding mounting lugs/loop).
- **Height:** < 15 mm populated height.
- **Weight target:** 30–60 g total including enclosure, battery, and all electronics.
- **Enclosure:** Jewelry-grade pendant enclosure. CAD files in `/mechanical/cad/pendant_enclosure.step`.
- **Water resistance:** IP54 minimum (sweat, rain); IP67 target (hand washing, brief immersion).
- **Charging:** USB-C or magnetic POGO pin (compatible with jewelry wear and water resistance).

---

## 8. Bill of Materials Overview

A full BOM is available in `hardware/bom/pendant_node_bom.csv`.

**Key Components Summary:**

| Ref | Part | Description |
|---|---|---|
| U1 | nRF5340-QKAA-AB0R | MCU |
| U2 | IWR6843ISK-ODS | mmWave Radar |
| U3 | ICM-42688-P | IMU |
| U4 | nPM1300 | PMIC |
| U5 | DRV2605L | Haptic Driver |
| MK1–MK3 | SPH0645LM4H-B | Microphones (PDM) |
| J1 | USB-C Connector | Power + Debug |
| SW1 | DPST Slide Switch | **Hardware Kill Switch — mandatory** |

---

## 9. Layout Guidelines

1. **RF Section:** mmWave radar antenna area must be free of copper and components on all layers. Follow manufacturer keep-out strictly.
2. **High-Speed Signals:** Route MIPI CSI traces with controlled impedance (100 Ω differential). Length-match the pairs within 5 mil.
3. **Power Decoupling:** Place 100 nF decoupling capacitors within 0.5 mm of all power pins. Add 10 µF bulk near PMIC.
4. **Grounding:** Solid ground plane on Layer 2. Star grounding prevents digital noise from coupling into mic analog inputs.
5. **Kill Switch Routing:** Route `V_CAM` rail through the switch contacts. Do not allow any alternative path that bypasses the switch.

---

## 10. Hardware Revision History

| Rev | Date | Description |
|---|---|---|
| v1.0 | 2026-05-05 | Initial reference design release |
