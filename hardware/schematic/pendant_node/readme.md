# Pendant / Necklace Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Neck/chest
**Primary Role:** Upper-hemisphere sensing, acoustic classification, visual identification (opt-in), 360° world capture (curved variant)
**Status:** Reference Design v1.0 — Three Variants
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture

**The pendant node has NO WiFi and NO cellular connectivity.**

All pendant data transmits via BAN (BLE 5.x or UWB) to the belt node exclusively. The belt node is the sole external network gateway.

**Data flow:**
```
Pendant sensors → BAN (BLE 5.x / UWB) → Belt Node → WiFi/Cellular → Companion App
```

If the user has configured live video streaming, the video is encoded on the pendant, transmitted via BAN to the belt node, and the belt node relays it over WiFi/cellular to the companion app.

---

## 2. Overview

The Pendant Node is the highest-information-value wearable node. Three variants exist:
- **Variant A — Standard Flat Pendant:** Compact medallion form, mmWave + IMU + acoustic, optional single camera
- **Variant B — 360° Curved Pendant:** Custom curved/rigid-flex PCB with distributed cameras for continuous 360° visual coverage
- **Variant C — Medallion (Premium):** Larger form with higher compute, more sensors, higher-capacity battery

---

## 3. Variant A — Standard Flat Pendant

### 3.1 Form Factor

- PCB: maximum 50 mm × 50 mm
- Populated height: < 15 mm
- Weight target: 30–60 g total
- Water resistance: IP54 minimum; IP67 target
- Charging: USB-C or magnetic POGO pin

### 3.2 Candidate Component Set (Test Variants — Not Locked)

#### MCU
- **Variant A:** Nordic nRF5340 (dual Cortex-M33, BLE 5.3, 128 MHz application core)
- **Variant B:** STM32WB55 (Cortex-M4 + M0+, BLE 5.0, ultra-low power)
- **Variant C:** NXP i.MX RT1060 (Cortex-M7, 600 MHz — for heavier local inference)

#### mmWave Radar
- **Variant A:** TI IWR6843ISK-ODS (60–64 GHz, evaluation module)
- **Variant B:** Acconeer XR112 (60 GHz, SPI, ultra-compact — preferred for wearable)
- **Variant C:** Infineon BGT60ATR24C (60 GHz, MMIC, custom antenna)
- **Variant D:** Acconeer XM125 (60 GHz, ultra-long range)

Coverage: antenna oriented for forward and lateral hemisphere from chest position.

#### IMU
- **Variant A:** Bosch BMI270 (6-axis, 0.1 mg noise, SPI — preferred)
- **Variant B:** TDK ICM-42688-P (6-axis, 0.07 mg noise, smallest package)
- **Variant C:** ST LSM6DSV (6-axis + temperature, iNEMO, built-in step/gesture)
- **Variant D:** Bosch BMI323 (6-axis, ultra-low power)

#### Microphone Array (Acoustic)

**3-element planar (compact):**
- Knowles SPH0645LM4H × 3 (I2S PDM, 65 dBA SNR)

**4-element tetrahedral (3D DOA):**
- ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR)
- InvenSense ICS-40180 × 4 (analog, 65 dBA SNR)

**6-element (high resolution):**
- Any above × 6 in circular or tetrahedral arrangement

#### Optional Camera Module (Opt-In)
- **Variant A:** OmniVision OV2640 (2 MP, MIPI/DVP)
- **Variant B:** Sony IMX307 (2 MP, low light, MIPI CSI-2)
- **Variant C:** Arducam IMX219 (8 MP, CSI)
- **Variant D:** OmniVision OV5640 (5 MP, autofocus)
- **Variant E:** HiMax HM01B0 (QVGA, 2.1 mW, on-chip first-stage detection)
- **Variant F:** Luxonis OAK-D Lite (stereo + neural inference on-module)

#### Optional Identification Modules (Research Variants)
- **Visual depth:** Intel RealSense D415
- **Thermal:** FLIR Lepton 3.5 (160×120)
- **Near-IR structured light:** Dot-projector + IR camera
- **Audio biometric:** Sensory TrulyHandsfree
- **Fingerprint:** Synaptics FS4500 (fixed entry points only)

#### Environmental Sensor
- **Variant A:** Bosch BME688 (temperature, humidity, pressure, VOC)
- **Variant B:** Sensirion SEN55 (PM2.5, PM10, temperature, humidity, VOC, NOx)

#### Optional Event Sensor
- **Variant A:** Prophesee Metavision EVK3 (640×480, USB)
- **Variant B:** iniVation DAVIS346 (346×260, combined frame+events)

#### Power Management
- Nordic nPM1300 PMIC (charging, regulated rails, fuel gauge)
- LiPo 3.7V, 150–300 mAh

#### Privacy Controls (User-Configured — None Mandatory)
- Hardware power switch (optional, user-installed): physically cuts V_CAM power rail
- Software scheduling: camera inactive during configured hours
- Detection-triggered mode: camera activates only on belt controller detection event
- Always-on mode: continuous recording per storage configuration
- Any combination supported

#### BAN Radio
- **Primary:** BLE 5.3 via MCU integrated radio
- **Optional:** Qorvo DW3000 UWB (precision time-sync + ranging; **NOT for external connectivity — BAN only**)

**Note:** UWB on the pendant is used for BAN communication and time-sync with the belt node only. It does not provide external network connectivity.

#### Haptic Actuator
- TI DRV2605L (I2C, waveform library)
- LRA: Precision Microdrives C10-100 (10 mm, 150 Hz)

---

## 4. Variant B — 360° Curved Pendant / Necklace

### 4.1 Purpose

Custom curved flexible or rigid-flex PCB following the necklace arc. Camera modules distributed at angular intervals. Produces continuous 360° visual coverage from chest level. All camera data transmits via BAN to belt node for stitching and relay.

### 4.2 Camera Coverage Configurations

| Config | Cameras | Angular Spacing | FoV Needed | Coverage |
|---|---|---|---|---|
| Minimal | 4 | 90° | ≥ 100° | 360° with overlap |
| Standard | 6 | 60° | ≥ 70° | 360° good overlap |
| Dense | 8 | 45° | ≥ 55° | 360° excellent overlap |
| Ultra | 12 | 30° | ≥ 40° | 360° maximum overlap |

### 4.3 PCB Construction Options

**Option A — Rigid-Flex:** Rigid PCB segments for components; flexible bridges between segments. Best reliability.

**Option B — Fully Flexible PCB:** Single substrate, multi-layer flex. Better conformability. More fragile.

**Option C — Modular Array:** Individual camera PCBs on flex ribbon cables. Most repairable.

**Option D — Curved Rigid PCB:** Standard rigid PCB bent during enclosure assembly (only for radii > 30 mm).

### 4.4 Camera Modules Per Position

- **Standard:** OV2640 (2 MP, DVP, tiny), OV5640 (5 MP, autofocus)
- **Low-light:** Sony IMX307 or IMX327
- **High-res:** Sony IMX219 (8 MP) or IMX477 (12 MP)
- **Ultra-low power:** HiMax HM01B0 (QVGA, first-stage detection on-chip)
- **Wide-angle:** Any above with fisheye or wide-angle optics

### 4.5 Image Processing Pipeline

**Level A — Belt Node Stitching (recommended for v1):**
- Raw camera streams transmitted via BAN high-bandwidth link to belt node
- Belt node Linux SoM runs stitching
- Real-time 360° equirectangular output
- Full-resolution stored on belt SD card
- Companion app accesses 360° view via WiFi from belt

**Level B — On-Pendant ISP:**
- Dedicated ISP (Allwinner V851 or similar) on pendant
- Stitches locally, transmits single compressed 360° stream via BAN to belt
- Reduces belt compute; increases pendant power

**Level C — Companion App Stitching:**
- Individual compressed camera streams from belt to companion app via WiFi
- App stitches at full quality
- Requires WiFi; belt relays all streams

### 4.6 MCU Architecture Options

**Option A — Single High-Performance MCU:**
- NXP i.MX 8M Mini (quad-core Cortex-A53 + Cortex-M4)
- Handles all camera multiplexing, compression, BAN coordination

**Option B — Two-MCU Split:**
- Sensing MCU: nRF5340 (BAN radio, IMU, radar, acoustic)
- Vision MCU: NXP i.MX RT1060 or Allwinner V851 (cameras, compression)
- Both communicate via SPI/UART internally

**Option C — Research Phase Minimal:**
- nRF5340 for radar/IMU/acoustic/BAN
- Camera streams to belt via USB (inelegant but functional for prototype)

### 4.7 Power

- Total active draw (all cameras + radar + IMU): 2–5 W
- Battery: 400–1000 mAh in pendant arc or separate module on chain
- Expected runtime (full 360° active): 2–4 hours research phase
- Belt power supplementation: conductive traces through necklace cable for extended runtime

### 4.8 Physical Design

- Curved enclosure: titanium or surgical steel channel
- Front face: optical-quality transparent polycarbonate at each camera position
- Inner surface: medical-grade silicone
- Weight: 50–120 g depending on battery configuration
- Balance: battery distributed along arc or in separate chain module

---

## 5. Variant C — Medallion (Premium)

- PCB: 60 mm × 60 mm or circular 60 mm diameter
- Thickness: 15–25 mm
- Higher compute: NXP i.MX 8M Plus or Raspberry Pi Zero 2W
- 6-element tetrahedral microphone array
- Higher-resolution camera (up to 12 MP)
- Integrated depth sensor or structured light
- Battery: 800–1500 mAh
- Weight: 80–150 g

---

## 6. Interface Summary

### Common Interfaces (All Variants)

| Interface | Signal Names | Connection |
|---|---|---|
| **mmWave Radar** | `SPI_MOSI/MISO/CLK`, `RADAR_CS`, `RADAR_IRQ` | SPI + GPIO |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI |
| **Microphone Array** | `PDM_CLK`, `PDM_DATA` | PDM or I2S |
| **Environmental** | `ENV_SDA`, `ENV_SCL` | I2C |
| **BAN Radio** | Integrated MCU | BLE 5.x (BAN to belt only) |
| **Charging** | `CHRG_IN+`, `CHRG_IN-` | POGO or USB-C |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |

### Variant A / C Additional

| Interface | Signal Names | Connection |
|---|---|---|
| **Camera** | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 |
| **Privacy switch (optional)** | `CAM_PWR_SW` | GPIO + MOSFET |
| **Haptic** | `HAPTIC_SDA`, `HAPTIC_SCL` | I2C |

### Variant B Additional (360° Curved)

| Interface | Signal Names | Connection |
|---|---|---|
| **Camera × N** | `CAM_n_MIPI_CLK+/-`, `CAM_n_D0+/-` | MIPI CSI-2 per camera |
| **Camera sync** | `CAM_FSYNC` | GPIO to all camera FSYNC |
| **Vision MCU link** | `USB3_TX/RX` or `SPI_HS` | High-speed to belt via BAN |
| **Belt power** | `V_BELT_IN` | From belt battery (optional) |

---

## 7. Power Architecture

### Variant A

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7–4.2V | LiPo | Radio PA, haptic |
| `V_SYS` | 3.3V | nPM1300 | MCU, sensors |
| `V_CAM` | 3.3V / 2.8V | Switched | Camera (user-configured) |

### Variant B (360°)

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7–4.2V | LiPo | All subsystems |
| `V_SYS` | 3.3V | Regulated | MCU, radar, IMU |
| `V_CAMS` | 3.3V / 2.8V | Regulated, ganged | All N cameras |
| `V_ISP` | 1.8V / 1.0V | Vision MCU supply | Vision MCU |
| `V_EXT` | 5V | From belt (optional) | Extended runtime |

---

## 8. 360° Pendant Manufacturing Notes

**PCB Stack (rigid-flex, 8-camera):**
- 8 rigid sections: each ~15 mm × 20 mm
- 7 flex bridges: each ~5 mm × 30 mm
- Minimum bend radius: 3 mm (polyimide)
- MIPI CSI-2 differential pairs: 100 Ω ± 10%
- Length-match within 2 mil per pair

---

## 9. Testing Strategy

### All Variants
1. Power-on self-test
2. IMU initialization and calibration check
3. Radar detection at 0.5 m
4. BAN connectivity to belt node (ping latency < 10 ms)
5. Battery and charging verification

### Variant A/C Camera
1. Camera initialization
2. Data handling mode verification per configured mode
3. Optional hardware switch test (if installed)

### Variant B 360°
1. All N cameras initialize within 5 seconds
2. FSYNC sync verified — all cameras trigger within ±1 ms
3. Stitching quality (belt node): straight lines continuous across boundaries
4. Full 360° recording test stored on belt SD card
5. Companion app 360° view accessible via belt WiFi within 5 seconds

---

## 10. Directory Structure

```
pendant_node/
├── variant_a_standard/
│   ├── pendant_node_std.kicad_pro
│   └── gerbers/
├── variant_b_360_curved/
│   ├── pendant_360_segment.kicad_pro
│   ├── pendant_360_flex_bridge.kicad_sch
│   ├── pendant_360_full_assembly.kicad_pcb
│   ├── pendant_360_stitching_config.json
│   └── gerbers/
├── variant_c_medallion/
│   ├── medallion_node.kicad_pro
│   └── gerbers/
├── bom/
├── 3d_models/
└── README.md
```
