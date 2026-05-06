# Bracelet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Wrist/forearm — worn on both left and right wrists
**Primary Role:** Forearm-hemisphere sensing, directional haptic output, optional visual capture
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Bracelet Node provides forearm-hemisphere sensing and the primary directional haptic alert output. Two nodes (left and right wrist) together cover lateral body hemispheres and discriminate forward/rear approaches.

All variants are camera-free by default. Variant C adds an optional outward-facing camera for users wanting arm-direction visual capture.

---

## 2. Design Philosophy

No artificial limits on sensor selection, storage capacity, or data transmission. The architecture is defined by its interface standards, not its components.

**Privacy controls:** Any camera installed on a bracelet node follows user-configured data handling policy. No system-imposed restrictions. Optional hardware power switch available to users who want physical camera disable.

---

## 3. Node Variants

### Variant A — Minimal

**Purpose:** Lowest weight, thinnest profile. Primary sensing with no auxiliary sensors.

**Sensors:**
- mmWave radar: Acconeer XR112 (60 GHz, SPI, ultra-compact)
- IMU: Invensense ICM-42688-P (smallest package, 6-axis)
- Haptic: LRA actuator (TI DRV2605L driver, 8–10 mm diameter LRA)

**Form factor:** Ultra-slim band (< 3 mm z-profile). Essentially invisible.

**Use case:** Stealth deployment. Minimal weight for comfort studies.

### Variant B — Standard

**Purpose:** Full radar capability with IMU and comfortable haptic. The recommended research variant.

**Sensors:**
- mmWave radar: Acconeer XR112 or TI IWR6843-class module
- IMU: Bosch BMI270 (6-axis, consistent with all other nodes)
- Haptic: LRA actuator (DRV2605L, C10-100)
- Optional: STMicro VL53L5CX ToF (8×8 zone, 6 m) for close-range geometry

**Form factor:** Standard bracelet thickness (5–8 mm). Watch-like profile.

**Use case:** Primary research platform. Full forearm coverage and directional haptics.

### Variant C — Extended (Camera)

**Purpose:** All Variant B capabilities plus optional outward-facing camera for arm-direction visual capture.

**Sensors:**
- All Variant B sensors
- Outward-facing camera module (one camera, faces away from wrist — in the direction the arm is pointing)
- Larger battery

**Camera options (all user-configured, no mandatory hardware switch):**
- **Option 1:** HiMax HM01B0 (QVGA, 2.1 mW, ultra-low power, on-chip detection)
- **Option 2:** OmniVision OV2640 (2 MP, DVP, compact)
- **Option 3:** Sony IMX219 (8 MP, CSI-2)

**Data handling:** Video recording to SD card, streaming to companion app, metadata-only — all per user configuration.

**Optional hardware privacy switch:** User can install a small hardware switch that physically cuts power to the camera module. This is an option, not a requirement. PCB provides the footprint and MOSFET for either hardware or software-only control.

**Form factor:** Slightly thicker dorsal profile (8–12 mm). Inner surface flat for comfort.

---

## 4. Candidate Component Set (All Variants — Not Locked)

### MCU

- **Variant A:** Silicon Labs EFR32BG24 (BLE 5.3, Cortex-M33, ultra-low power)
- **Variant B:** Nordic nRF5340 (BLE 5.3, dual Cortex-M33 — consistent with pendant for firmware uniformity)
- **Variant C:** STM32WL55 (BLE + LoRa, Cortex-M4, ultra-low power)

### mmWave Radar

- **Variant A:** Acconeer XR112 (60 GHz, SPI, 30 mW, ultra-compact) — preferred for wrist
- **Variant B:** Infineon BGT60ATR24C (60 GHz, MMIC, custom antenna design)
- **Variant C:** TI IWR6843AOP (60 GHz, AOA, higher angular resolution — bulkier)

**Antenna orientation:** PCB antenna faces outward (palm/dorsal side). Maximizes coverage of the hemisphere ahead of and to the side of the arm.

### IMU

- **Variant A:** Bosch BMI270 (6-axis, 0.1 mg noise, SPI — preferred for consistency)
- **Variant B:** TDK ICM-42688-P (0.07 mg noise, smallest package)
- **Variant C:** ST LSM6DSV (6-axis + temperature, built-in gesture/step)

### Haptic Actuator

- **Variant A:** Precision Microdrives C10-100 LRA (10 mm, 150 Hz resonance) — preferred
- **Variant B:** Precision Microdrives 307-103 ERM (simpler driver, wider frequency)
- **Variant C:** AAC Technologies ALPS HF2-C3 (thin LRA, 2 mm z-height — for Variant A minimal)

**Driver IC:** TI DRV2605L (I2C, waveform library, auto-resonance via back-EMF). Mandatory for LRA. Optional for ERM.

### Optional Short-Range ToF (Variant B)

- STMicro VL53L5CX (8×8, 6 m, I2C) — close-range geometry where radar resolution insufficient

### Camera Module (Variant C Only)

See Variant C description. All options above are valid. No camera is installed on Variants A or B by default.

### BAN Radio

- **Primary:** BLE 5.3 via MCU integrated radio
- **Optional:** Qorvo DW3000 UWB (sub-ns time-sync + ranging — research variant)

### Battery

- Variant A: LiPo 3.7V, 150–250 mAh (ultra-slim)
- Variant B: LiPo 3.7V, 200–400 mAh (standard)
- Variant C: LiPo 3.7V, 300–500 mAh (extended for camera)

### Charging

- Magnetic pogo-pin contacts on inner wrist surface (skin-contact side)
- Optional: Qi wireless charging coil (thicker band required)

---

## 5. Bracelet-Specific Design Notes

**Form factor:** PCB in dorsal (back-of-hand) facing band section. Sensor apertures face outward. Must not impede wrist flexion.

**Antenna design:** mmWave at 60 GHz: PCB trace antenna or small patch. Must face outward. Body absorption significantly attenuates signals facing the wrist — outward orientation is mandatory for useful range.

**Water resistance:** IP67 mandatory. Sweat, rain, hand washing guaranteed.

**Material contact:** EN 1811:2011 nickel release compliance for all skin-contact surfaces. Hypoallergenic materials required (grade 5 titanium, 316L surgical steel, medical-grade silicone, hard-coat anodized aluminum).

**Left vs Right:** Hardware identical. `bracelet_left.rs` and `bracelet_right.rs` firmware binaries handle coordinate mapping and haptic direction encoding.

---

## 6. Interface Summary

| Interface | Signal Names | Connection |
|---|---|---|
| **mmWave Radar** | `SPI_MOSI/MISO/CLK`, `RADAR_CS`, `RADAR_IRQ` | SPI + GPIO |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI (shared bus, separate CS) |
| **Haptic Driver** | `HAPTIC_SDA`, `HAPTIC_SCL`, `HAPTIC_EN` | I2C + GPIO |
| **Optional ToF** | `TOF_SDA`, `TOF_SCL`, `TOF_INT` | I2C |
| **Camera (Var C)** | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 |
| **Optional HW Switch (Var C)** | `CAM_HW_SW` | GPIO → MOSFET |
| **BAN Radio** | Integrated in MCU | — |
| **Charging** | `CHRG_IN+`, `CHRG_IN-` | Pogo pins |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |

---

## 7. Directory Structure

```
bracelet_node/
├── variant_a_minimal/
│   ├── bracelet_minimal.kicad_pro
│   └── gerbers/
├── variant_b_standard/
│   ├── bracelet_std.kicad_pro
│   └── gerbers/
├── variant_c_extended_camera/
│   ├── bracelet_ext.kicad_pro
│   └── gerbers/
├── bom/
└── README.md
```
