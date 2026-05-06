# Pendant / Necklace Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Neck/chest — worn as a necklace, pendant, or medallion
**Primary Role:** Upper-hemisphere sensing, acoustic classification, visual identification (opt-in), 360° world capture (curved variant)
**Status:** Reference Design v1.0 — Three Variants
**License:** CERN-OHL-S v2

---

## 1. Overview

The Pendant Node is the highest-information-value wearable node. Its chest/neck position gives it 360° azimuthal view of the environment at chest height — the most relevant sensing plane for approaching humans and objects. It also hosts the primary acoustic sensor array and the optional visual identification capability.

Three distinct physical variants exist, ranging from a minimal flat pendant to a fully custom curved 360° necklace with distributed cameras around the circumference. All variants share the same firmware foundation and BAN protocol, differing only in physical form and sensor population.

---

## 2. Variant A — Standard Flat Pendant

### 2.1 Purpose

Flat PCB in a medallion-style enclosure. Optimized for minimal size and weight while providing mmWave sensing, IMU, and acoustic classification. Optional single-camera forward-facing module. The most production-feasible variant for early research phases.

### 2.2 Form Factor Constraints

- PCB: maximum 50 mm × 50 mm
- Populated height: < 15 mm
- Weight target: 30–60 g total (including enclosure and battery)
- Water resistance: IP54 minimum; IP67 target
- Charging: USB-C or magnetic POGO pin

### 2.3 Candidate Component Set (Test Variants — Not Locked)

#### MCU
- **Variant A:** Nordic nRF5340 (dual Cortex-M33, BLE 5.3, 128 MHz application core)
- **Variant B:** STM32WB55 (Cortex-M4 + M0+, BLE 5.0, ultra-low power)
- **Variant C:** NXP i.MX RT1060 (Cortex-M7, 600 MHz — for heavier local inference)

#### mmWave Radar
- **Variant A:** TI IWR6843ISK-ODS (60–64 GHz, evaluation module, initial research)
- **Variant B:** Acconeer XR112 module (60 GHz, SPI, ultra-compact — preferred for wearable)
- **Variant C:** Infineon BGT60ATR24C (60 GHz, MMIC, requires custom antenna layout)
- **Variant D:** Acconeer XM125 (60 GHz, ultra-long range, UART)

Coverage: Antenna oriented for forward and lateral hemisphere from chest position.

#### IMU
- **Variant A:** Bosch BMI270 (6-axis, 0.1 mg noise density, SPI — preferred)
- **Variant B:** TDK ICM-42688-P (6-axis, 0.07 mg noise density, smallest package)
- **Variant C:** ST LSM6DSV (6-axis + temperature, iNEMO, built-in step/gesture)
- **Variant D:** Bosch BMI323 (6-axis, ultra-low power, advanced motion processing)

#### Microphone Array (Acoustic)
- **3-element planar (compact):**
  - Knowles SPH0645LM4H × 3 (I2S PDM, 65 dBA SNR)
- **4-element tetrahedral (3D DOA):**
  - ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR) on flexible PCB
  - InvenSense ICS-40180 × 4 (analog, 65 dBA SNR)
- **6-element (high resolution):**
  - Any above × 6 in circular or tetrahedral arrangement

#### Optional Camera Module (Opt-In Single Camera)
- **Variant A:** OmniVision OV2640 (2 MP, MIPI/DVP, small package)
- **Variant B:** Sony IMX307 (2 MP, low light, MIPI CSI-2)
- **Variant C:** Arducam IMX219 (8 MP, CSI, configurable optics)
- **Variant D:** OmniVision OV5640 (5 MP, autofocus)
- **Variant E:** HiMax HM01B0 (QVGA, 2.1 mW, ultra-low power, on-chip first-stage detection)
- **Variant F:** Luxonis OAK-D Lite (stereo + neural inference on-module)

#### Optional Visual Identification Modules (All Research Variants)
- **Visual (depth):** Intel RealSense D415 (stereo depth + RGB)
- **Thermal:** FLIR Lepton 3.5 (160×120 thermal imaging)
- **Near-IR structured light:** Dot-projector + IR camera
- **Audio biometric:** Sensory TrulyHandsfree (voice recognition pre-processor)
- **Fingerprint:** Synaptics FS4500 (fixed entry points only)

#### Environmental Sensor
- **Variant A:** Bosch BME688 (temperature, humidity, pressure, VOC — single chip)
- **Variant B:** Sensirion SEN55 (PM2.5, PM10, temperature, humidity, VOC, NOx)

#### Optional Event Sensor (Fast Transient)
- **Variant A:** Prophesee Metavision EVK3 (640×480, USB)
- **Variant B:** iniVation DAVIS346 (346×260, combined frame+events)

#### Power Management
- **PMIC:** Nordic nPM1300 (charging, regulated rails, fuel gauge)
- **Battery:** LiPo 3.7V, 150–300 mAh (jewelry weight budget)
  - 150 mAh: ultra-thin 1.5 mm profile
  - 200 mAh: standard pendant thickness
  - 300 mAh: bulkier pendant, 5 mm profile

#### Privacy Controls (All User-Configured — None Mandatory)
- **Hardware power switch (optional):** User-installed switch cutting V_CAM power rail. Recommended for users wanting guaranteed physical camera disable without software interaction.
- **Software scheduling:** Camera driver disabled in firmware when schedule policy says inactive.
- **Activity-trigger mode:** Camera only activates on belt controller detection event.
- **Always-on mode:** Continuous recording per storage configuration.
- **Any combination** of the above is supported.

No specific privacy control method is mandated by the architecture. Users select what suits their use case and jurisdiction.

#### BAN Radio
- **Primary:** BLE 5.3 via MCU integrated radio
- **Optional:** Qorvo DW3000 UWB (precision time-sync + ranging)

#### Haptic Actuator
- TI DRV2605L (I2C, waveform library)
- LRA actuator: Precision Microdrives C10-100 (10 mm diameter, 150 Hz)

---

## 3. Variant B — 360° Curved Pendant / Necklace

### 3.1 Purpose

This is the signature advanced variant of SENTINEL-WEAR. A custom curved PCB (flexible or rigid-flex) follows the arc of the necklace pendant, with camera modules distributed at regular angular intervals around the circumference. Together they produce continuous 360° visual coverage from the chest level — functionally equivalent to a full 360° security camera array, but mobile, wearable, and body-frame aware.

### 3.2 Physical Architecture

The necklace hangs from a chain or cable at chest level. The pendant element is an arc-shaped PCB spanning approximately 180°–360° of the pendant face, depending on the chosen camera coverage configuration.

**Coverage configurations:**

| Config | Cameras | Angular Spacing | Individual Camera FoV Needed | Coverage |
|---|---|---|---|---|
| Minimal | 4 | 90° | ≥ 100° | 360° with overlap |
| Standard | 6 | 60° | ≥ 70° | 360° with good overlap |
| Dense | 8 | 45° | ≥ 55° | 360° with excellent overlap |
| Ultra | 12 | 30° | ≥ 40° | 360° with maximum overlap |

**Front-face cameras:** Face outward from pendant surface (forward hemisphere).
**Side cameras:** Face perpendicular to pendant surface (lateral coverage).
**Rear cameras:** Face inward toward the wearer's body or outward from pendant rear (rear hemisphere). The rear-facing cameras may be placed on the chain/cable element rather than the pendant face.

### 3.3 PCB Construction

**Option A — Rigid-Flex**
- Rigid PCB segments for component placement
- Flexible bridges between segments
- Best for component density and reliability
- Segments: each camera module on its own rigid section
- Flex bridges: power, data, and timing lines between segments
- Total PCB arc: 8–12 rigid segments connected by 7–11 flex bridges

**Option B — Fully Flexible PCB**
- Single substrate, multi-layer flexible PCB
- Better conformability
- More fragile — vibration and flex fatigue concerns
- Suitable for low-bend-cycle deployment

**Option C — Modular Array**
- Individual camera modules on small PCBs
- Connected by thin flexible ribbon cables
- Maximum repairability and component substitution
- Mechanically simpler but bulkier per connection

**Option D — Curved Rigid PCB with Formed Enclosure**
- Standard rigid PCB bent during enclosure assembly (only viable for gentle curves)
- Simpler to manufacture
- Only for radii > 30 mm

### 3.4 Candidate Camera Modules (Per-Position)

Each position on the arc receives one camera module. All modules at all positions are the same type for simplest BOM, though mixed configurations are supported.

- **Standard Quality:** OV2640 (2 MP, DVP, tiny), OV5640 (5 MP, autofocus)
- **Low-Light:** Sony IMX307 or IMX327 (2 MP, excellent low-light sensitivity)
- **High Resolution:** Sony IMX219 (8 MP) or IMX477 (12 MP)
- **Ultra-Low Power:** HiMax HM01B0 (QVGA, first-stage detection on-chip) — for less active positions
- **Wide-Angle:** Custom wide-angle lens fitted to any above module
- **Fisheye:** 180° fisheye lens on any above module for maximum per-camera coverage

Recommended for initial research: OmniVision OV2640 at all positions (affordable, small, adequate quality) with fisheye or wide-angle optics.

### 3.5 Image Processing Pipeline — 360° Stitching

**Level A — Belt Node Processing (recommended for v1)**
- Raw streams from all cameras transmitted over high-bandwidth link to belt node
- Belt node Linux SoM runs stitching software
- Real-time 360° equirectangular output at reduced resolution
- Full-resolution recorded to SD card on belt

**Level B — Distributed On-Pendant ISP**
- Dedicated ISP chip on pendant (e.g., Allwinner V851 or similar)
- Performs stitching from all camera streams locally
- Compressed 360° stream transmitted to belt node
- Reduces belt node compute requirement
- Increases pendant power and complexity

**Level C — Companion App Processing**
- All raw streams transmitted to companion app over Wi-Fi
- App performs stitching at full quality
- Enables highest quality output
- Requires continuous Wi-Fi connectivity

**Level D — Cloud Offload (Optional)**
- Compressed streams transmitted to user's own server or cloud storage
- Stitching and object recognition in cloud
- Highest quality, highest latency, requires internet

### 3.6 Image Stitching Software

**On belt node (Linux SoM):**
- `ffmpeg` with overlay filter for initial calibration
- OpenCV-based custom stitcher (per-camera calibration + homography + blend)
- Or: commercial 360° stitching SDK (Kandao, Insta360 developer tools)

**On companion app:**
- GPU-accelerated stitching via Metal (iOS) / Vulkan (Android) / DirectX/Metal (desktop)

**Calibration:** Per-camera intrinsics + extrinsics. Calibrated once at manufacturing/setup via standard checkerboard pattern at multiple distances. Calibration data stored on belt node SD card.

### 3.7 SLAM Integration

The 360° pendant is an extremely powerful SLAM sensor when combined with the belt node:

- 360° visual input from pendant cameras provides omnidirectional visual odometry
- Combined with anklet ToF ground-plane data and belt IMU torso reference
- Produces a continuously-updated 3D map anchored to the body
- As the wearer moves, the entire visited environment is mapped in 3D
- The SLAM map persists as a record of everywhere the wearer has been with the geometry of those spaces

**SLAM stack:**
- **Backend:** ORB-SLAM3 (CPU-based) or RTK-SLAM (GPU if available)
- **Visual odometry:** Per-camera feature tracking + fusion across all 360° cameras
- **Dense reconstruction:** OpenMVS or similar for mesh generation from keyframes
- **Object recognition:** YOLO or equivalent running on belt node or companion app

### 3.8 360° Pendant MCU Architecture

Due to the data bandwidth from multiple cameras, the 360° pendant requires a more capable compute structure:

**Option A — Single High-Performance MCU**
- NXP i.MX 8M Mini (quad-core Cortex-A53 + Cortex-M4)
- Handles: camera multiplexing, compression, IMU, radar, BAN radio coordination
- Transmits compressed streams to belt node

**Option B — Two-MCU Split**
- Sensing MCU: nRF5340 (BAN radio, IMU, radar, acoustic)
- Vision MCU: NXP i.MX RT1060 or Allwinner V851 (camera multiplexing, initial compression)
- Vision MCU connected to belt node via USB 3.x or high-speed proprietary link

**Option C — Minimal (research phase)**
- nRF5340 handles radar, IMU, acoustic, BAN
- Each camera connected directly to belt node via separate USB cables (inelegant but functional for initial testing)

### 3.9 Power Architecture

- **Total draw (all cameras active, radar + IMU):** 2–5 W
- **Battery:** 400–1000 mAh distributed in pendant arc or in a separate battery module worn on chain/cable
- **Charging:** Magnetic pogo-pin dock or Qi wireless
- **Expected runtime (full 360° active):** 2–4 hours (research phase); 6–12 hours (optimized production)

### 3.10 Physical Design Notes

**Jewelry aesthetic:**
- The curved PCB arc is enclosed in a jewelry-grade curved enclosure
- Front face: optical-quality transparent polymer or sapphire glass at each camera position
- Frame: titanium, surgical steel, or anodized aluminum
- Camera apertures: tiny holes or transparent windows aligned to each camera module
- Back surface: medical-grade silicone for comfort against skin

**Weight distribution:**
- Larger battery modules can be distributed to the chain/cable for balance
- The pendant itself should weigh no more than the heaviest luxury pendant jewelry (~50–80 g)
- Battery extension module on the chain adds capacity without increasing pendant weight

**Necklace chain:**
- Standard jewelry chain or cable provides mechanical support
- Optional: conductive traces embedded in cable for power routing from belt battery bank

---

## 4. Variant C — Medallion (Premium / High-Capability)

### 4.1 Purpose

A larger, thicker pendant form factor. Sacrifices minimum form factor for maximum sensing capability. Suitable for use cases where sensing quality is the top priority.

### 4.2 Key Differentiators from Variant A

- **Larger PCB:** 60 mm × 60 mm or circular 60 mm diameter
- **Thicker:** 15–25 mm populated height
- **Higher compute:** NXP i.MX 8M Plus or Raspberry Pi Zero 2W class
- **More microphones:** 6-element tetrahedral array (highest acoustic precision)
- **Higher-resolution camera:** Up to 12 MP with autofocus
- **Integrated short-range structured light or depth sensor:** True 3D face recognition capability
- **Higher-capacity battery:** 800–1500 mAh
- **Weight:** 80–150 g (still within pendant comfort range for most users)

### 4.3 Use Case

Medical monitoring, high-fidelity research collection, extreme accuracy requirements, users who prioritize capability over minimalism.

---

## 5. Interface Summary (All Variants)

### Common Interfaces

| Interface | Signal Names | Connection |
|---|---|---|
| **mmWave Radar** | `SPI_MOSI/MISO/CLK`, `RADAR_CS`, `RADAR_IRQ` | SPI + GPIO |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI (shared bus, separate CS) |
| **Microphone Array** | `PDM_CLK`, `PDM_DATA` or `I2S_BCLK/WS/DATA` | PDM or I2S |
| **Environmental** | `ENV_SDA`, `ENV_SCL` | I2C |
| **BAN Radio** | Integrated in MCU or separate module | SPI/UART |
| **Charging** | `CHRG_IN+`, `CHRG_IN-` | POGO or USB-C |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |

### Variant A / C Additional

| Interface | Signal Names | Connection |
|---|---|---|
| **Camera (single)** | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 |
| **Optional privacy switch** | `CAM_PWR_SW` | GPIO + MOSFET |
| **Haptic** | `HAPTIC_SDA`, `HAPTIC_SCL` | I2C (DRV2605L) |

### Variant B Additional (360° Curved)

| Interface | Signal Names | Connection |
|---|---|---|
| **Camera × N (per module)** | `CAM_n_D0+/-`, `CAM_n_CLK+/-` | MIPI CSI-2 per camera |
| **Camera sync** | `CAM_FSYNC` (hardware sync line) | GPIO broadcast to all cameras |
| **Vision MCU** | `USB3_TX/RX` or `SPI_HS` | High-speed link to belt node |
| **Belt power** | `V_BELT_IN` | High-current from belt battery bank (optional) |

---

## 6. Power Architecture

### Variant A

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7–4.2V | LiPo | Radio PA, haptic peak |
| `V_SYS` | 3.3V | nPM1300 | MCU, sensors, radio |
| `V_CAM` | 3.3V or 2.8V | Switched | Camera (user-configured enable) |

### Variant B (360°)

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7–4.2V | LiPo | Power to all subsystems |
| `V_SYS` | 3.3V | Regulated | MCU, radar, IMU |
| `V_CAMS` | 3.3V / 2.8V | Regulated, ganged | All camera modules (single rail for all) |
| `V_ISP` | 1.8V / 1.0V | Vision MCU supply | Vision MCU core/IO |
| `V_EXT` | 5V | From belt node (optional) | Belt-supplied power for extended runtime |

---

## 7. 360° Pendant Manufacturing Notes

**PCB Stack (rigid-flex, 8-camera variant):**
- 8 rigid sections: each ~15 mm × 20 mm carrying one camera module
- 7 flex bridges: each ~5 mm × 30 mm carrying power + 4-lane MIPI CSI-2 + sync
- Total arc length: ~180 mm (matching pendant circumference)
- Minimum bend radius for flex sections: 3 mm (polyimide substrate)

**Impedance control:**
- MIPI CSI-2 differential pairs: 100 Ω ± 10%
- Match length within 2 mil on all MIPI pairs
- Ground plane continuity must be maintained across all flex bridges

**Assembly:**
- Rigid sections assembled independently on standard SMT line
- Flex bridges attached after individual section test
- Full flex assembly requires specialized bending fixtures

---

## 8. Directory Structure

```
hardware/schematic/pendant_node/
├── variant_a_standard/
│   ├── pendant_node_std.kicad_pro
│   ├── pendant_node_std.kicad_sch
│   ├── pendant_node_std.kicad_pcb
│   └── gerbers/
├── variant_b_360_curved/
│   ├── pendant_360_segment.kicad_pro     # Single arc segment (replicated ×N)
│   ├── pendant_360_flex_bridge.kicad_sch
│   ├── pendant_360_full_assembly.kicad_pcb
│   ├── pendant_360_stitching_config.json # Camera calibration and stitching params
│   └── gerbers/
├── variant_c_medallion/
│   ├── medallion_node.kicad_pro
│   ├── medallion_node.kicad_sch
│   ├── medallion_node.kicad_pcb
│   └── gerbers/
├── bom/
│   ├── pendant_std_bom.csv
│   ├── pendant_360_bom.csv
│   └── medallion_bom.csv
├── 3d_models/
│   ├── pendant_std_enclosure.step
│   ├── pendant_360_arc_segment.step
│   ├── pendant_360_full_assembly.step
│   └── medallion_enclosure.step
└── README.md
```

---

## 9. Testing Strategy

### All Variants

1. Power-on self-test
2. IMU initialization and calibration check
3. Radar detection at 0.5 m test target
4. BAN radio connectivity
5. Battery and charging circuit verification

### Variant A / C Camera Testing

1. Camera initialization response
2. Image quality check (resolution, focus at 1 m)
3. Privacy control verification per configured method (if hardware switch present: confirmed zero current to camera; if software mode: confirmed camera inactive)

### Variant B 360° Camera Testing

1. All N cameras initialize and produce frames
2. Hardware sync signal verified across all cameras
3. Stitching quality check: straight line continuity across camera boundaries
4. Full 360° recording test: 30-second clip stored to SD card
5. Companion app stream: live 360° view accessible within 5 seconds of activation

---

## 10. Related Documentation

- **Firmware:** `firmware/src/bin/pendant_node.rs`
- **Body-Frame Theory:** `docs/theory/body_coordinate_fusion.md`
- **360° Stitching:** `crates/sentinel-slam/src/stitching.rs` (when implemented)
- **SLAM Architecture:** `crates/sentinel-slam/src/lib.rs`
- **Hardware Config:** `hardware/hardware_config.md`
