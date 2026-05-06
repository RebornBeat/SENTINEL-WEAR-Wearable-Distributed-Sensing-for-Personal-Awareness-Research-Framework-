# Anklet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Lower leg / above ankle — worn on both left and right ankles
**Primary Role:** Ground-plane sensing, gait analysis, lower-hemisphere coverage, haptic alerts
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Anklet Node is the ground-level awareness and gait analysis layer. Its position on the lower leg provides:
- Detection of floor-level objects, obstacles, and approaching bodies from below
- Primary input for gait analysis (step detection, heel-strike, stumble precursor)
- Haptic alerts for ground-level threats and rear/low directional cues
- Optional downward camera for ground-level visual coverage

The high-motion profile of anklet nodes during walking makes them the best calibration anchors for walk-through body-frame calibration.

---

## 2. Node Variants

### Variant A — Minimal

**Purpose:** Lowest weight and profile. Core gait sensing and ground detection.

**Sensors:**
- STMicro VL53L5CX ToF (8×8 zone, 6 m range) — ground clearance and obstacle detection
- Bosch BMI270 IMU (6-axis) — gait analysis, step detection, heel-strike
- LRA haptic actuator (DRV2605L driver)

**Form factor:** Compact band. Target < 80 g total.

**Use case:** Gait analysis research. Basic ground-plane awareness.

### Variant B — Extended

**Purpose:** Full ground-level capability with optional camera for ground-level visual capture.

**Sensors:**
- Short-range LiDAR (3D point cloud, higher resolution than 1D ToF)
- Optional: Acconeer XR112 mmWave (forward-facing, detects approaching ground-level objects)
- Bosch BMI270 IMU
- LRA haptic actuator
- Optional: Downward-facing camera (for ground-level recording)
- Environmental sensor (humidity, temperature — context for gait analysis on different surfaces)

**Camera orientation:** Small camera module facing downward or forward-downward. Captures floor-level approaching objects, ground surface, and near-ground obstacles. Useful for stair detection, surface classification (carpet vs. tile — affects gait parameters).

**Data handling (camera):** Fully user-configured per `sentinel-wear.toml`.

---

## 3. Candidate Component Set (All Variants — Not Locked)

### MCU

- **Variant A:** Silicon Labs EFR32BG24 (BLE 5.3, ultra-low power, Cortex-M33)
- **Variant B:** Nordic nRF5340 (dual Cortex-M33, BLE 5.3, consistent firmware)
- **Variant C:** STM32L4 (ultra-low power, extended ADC for surface classification)

### Short-Range Sensing

**ToF (Variant A):**
- **Primary:** STMicro VL53L5CX (8×8 zone matrix, 6 m range, I2C) — preferred
- **Alternative:** Terabee TeraRanger Evo Mini (lightweight, UART)

**Short-range LiDAR (Variant B):**
- **Option A:** TFMini-S (UART, 12 m range, compact)
- **Option B:** Benewake TF-Luna (UART/I2C, 8 m, ultra-compact)

**mmWave (Variant B — optional):**
- **Option A:** Acconeer XR112 (60 GHz, SPI, forward-facing, ultra-low power)
- **Option B:** Acconeer XR111 (similar, evaluation module for early testing)

### IMU

- **Variant A:** Bosch BMI270 (6-axis, ultra-low power, SPI — consistent across all nodes)
- **Variant B:** ST LSM6DSV (6-axis + iNEMO, built-in ML for step detection)

IMU at ankle is the highest-dynamic-range node in the system — must handle:
- Heel-strike impacts (5–10 g peak)
- Large angular excursions during leg swing
- Running and sprinting at up to 5 g sustained

### Haptic Actuator

- **Variant A:** Precision Microdrives C10-100 LRA (10 mm, 150 Hz) — consistent with bracelet
- **Variant B:** ERM motor (cheaper, simpler driver, slightly less precise pattern control)

**Driver:** TI DRV2605L (I2C) — preferred. LRA resonance tracking via back-EMF.

### Camera Module (Variant B — Optional)

- **Option A:** HiMax HM01B0 (QVGA, 2.1 mW, ultra-low power) — lowest power, suitable for ankle
- **Option B:** OmniVision OV2640 (2 MP, DVP, wide-angle lens) — standard resolution
- **Option C:** Sony IMX219 miniaturized module with fisheye lens for wider floor coverage

**Orientation:** Downward-facing (floor and approaching ground obstacles) or forward-downward (approaching persons from below knee level).

**Optional hardware privacy switch:** User-installed if desired. PCB provides footprint. Not architecturally required.

### BAN Radio

- BLE 5.3 via MCU integrated radio (primary)
- UWB via Qorvo DW3000 (optional, for precise time-sync — particularly valuable at anklets where stride timing accuracy matters)

### Battery

- Variant A: LiPo 3.7V, 300 mAh
- Variant B: LiPo 3.7V, 400–500 mAh (larger to cover camera if installed)
- Target runtime: 48–72 hours Variant A; 24–48 hours Variant B

### Charging

- Qi wireless charging or magnetic pogo-pin dock
- Left and right anklets have identical charging accessories

---

## 4. Gait Analysis Integration

Anklet IMU data feeds `sentinel-body-frame::gait_analyzer`:

- **Step frequency (cadence):** Steps per minute from heel-strike interval
- **Step regularity:** Gait coefficient of variation — lower = more regular
- **Heel-strike impact:** Peak G at each strike — elevated values indicate hard surfaces or anomalous motion
- **Stride asymmetry:** Left vs. right foot comparison — asymmetry > threshold triggers anomaly alert
- **Pre-stumble signature:** Asymmetric loading + balance recovery micro-accelerations → `StumblePrecursor` event
- **Gait phase:** Stance / swing / strike / push — at 200 Hz IMU rate

These feed PentaTrack's `WearerSelfMotion` anomaly class and generate predictive stumble alerts.

---

## 5. Interface Summary

| Interface | Signal Names | Connection |
|---|---|---|
| **ToF / LiDAR** | `TOF_SDA`, `TOF_SCL`, `TOF_INT` | I2C or UART |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI |
| **mmWave (Var B opt)** | `MMWAVE_CS`, `MMWAVE_IRQ` | SPI |
| **Haptic** | `HAPTIC_SDA`, `HAPTIC_SCL` | I2C |
| **Camera (Var B opt)** | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 or DVP |
| **BAN Radio** | Integrated in MCU | — |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |
| **Charging** | `CHRG_IN+`, `CHRG_IN-` | Qi coil or pogo pins |

---

## 6. Mechanical Design

**Placement:** Worn on lower leg, above the ankle bone (malleolus). Medial or lateral mounting — calibration handles both. Left and right nodes are hardware-identical; firmware handles coordinate convention.

**Water resistance:** IP67 mandatory. Sweat, rain, mud are guaranteed exposure.

**Material contact:** EN 1811 nickel release compliance. Medical-grade silicone band or titanium/surgical steel clasp.

**Impact resistance:** Enclosure must survive:
- Heel-strike shock (10 g, 1 ms impulse)
- Walking into low furniture
- Running on hard surfaces

---

## 7. Directory Structure

```
anklet_node/
├── variant_a_minimal/
│   ├── anklet_minimal.kicad_pro
│   └── gerbers/
├── variant_b_extended/
│   ├── anklet_extended.kicad_pro
│   └── gerbers/
├── bom/
└── README.md
```
