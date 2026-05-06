# Bracelet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Wrist/forearm — worn on both left and right wrists
**Primary Role:** Forearm-frame distributed sensing + Haptic alert output
**Form Factor:** Jewelry-grade wearable (wrist/forearm mount)
**Status:** Reference Design

---

## 1. Purpose

The Bracelet Node is one of the primary sensing and interaction nodes in the SENTINEL-WEAR mesh. Worn on the wrist or forearm, it provides:

- **Sensing:** Forward and lateral hemisphere coverage relative to the wearer's arm.
- **Body-Frame Contribution:** The node's IMU data feeds the multi-IMU body-pose estimator, contributing to overall body-frame stabilization.
- **Alerting:** The primary haptic actuator for directional alerts. When an object approaches from the wearer's left side, the left bracelet fires the haptic pattern.

Two bracelet nodes (left and right wrist) together cover the lateral body hemispheres and improve forward/rear directional discrimination beyond what the pendant alone achieves.

---

## 2. Design Philosophy: Limitless Configuration

This architecture imposes no limits on the specific components used. The following describes functional categories only. Sensor resolution, range, update rate, power budget, and form factor are all configuration choices.

- **No Artificial Limits:** Interfaces support standard protocols at maximum defined speeds.
- **Modularity:** Each functional block (Sensing, Compute, Radio, Power) is defined by its interface, not a specific part number.
- **No Identification Layer:** The Bracelet Node is sensing-only. It is camera-free by architecture. No hardware kill-switch for an identification sensor is required. A power switch is recommended for user control.

---

## 3. Candidate Component Set (Test Variants — Not Locked)

### MCU

- **Variant A:** Nordic nRF5340 (Cortex-M33 app + M33 net, BLE 5.3, USB, consistent with pendant development)
- **Variant B:** Silicon Labs EFR32BG24 (BLE 5.3, ultra-low power, Cortex-M33)

Bracelet MCU compute requirements are lower than pendant. Primary tasks: radar polling, IMU Madgwick filter, haptic control, BAN radio.

### mmWave Radar

- **Variant A:** Acconeer XR112 (60 GHz, SPI, low power, compact) — preferred for wrist form factor
- **Variant B:** Infineon BGT60ATR24C with custom 2-element patch antenna

**Antenna orientation:** PCB antenna arrays oriented toward the palm/dorsal face of the wrist to maximize coverage of the hemisphere in front of and to the side of the bracelet-wearing arm.

### IMU

- **Variant A:** Bosch BMI270 (consistent with pendant — uniform calibration across nodes)
- **Variant B:** Invensense ICM-42688-P (smallest package option)

### Haptic Actuator

- **Variant A:** Precision Microdrives C10-100 linear resonant actuator (LRA, 10 mm diameter, 150 Hz resonance) — preferred
- **Variant B:** Precision Microdrives 307-103 (ERM, vibration motor, wider frequency range)
- **Variant C:** AAC Technologies ALPS HF2-C3 (thin LRA, 2 mm z-height, for ultra-slim bracelet)

LRA preferred over ERM for distinct directional patterns and lower power. Haptic driver IC: Texas Instruments DRV2605L (I2C, waveform library, auto-resonance detection, back-EMF feedback).

### Short-Range LiDAR / ToF (Optional)

- For close-range geometry refinement where radar resolution is insufficient.
- **Variant A:** STMicro VL53L5CX (8×8 matrix, 6 m range, I2C)

### BAN Radio

- BLE 5.3 via MCU integrated radio (primary)
- UWB via Qorvo DW3000 (optional, research variant for sub-millisecond sync)

### Battery

- LiPo 3.7V, 200–400 mAh
- Target 24–72 hour runtime between charges

### Charging

- Magnetic pogo-pin charging contacts on inner wrist surface (skin-contact side)
- Alternative: Qi wireless charging coil (thicker band required)

---

## 4. Architectural Capabilities

### Sensing

- **mmWave Radar:** Primary modality for detecting presence, velocity (Doppler), and micro-Doppler activity signatures (human vs. animal vs. vehicle). Illuminates forearm-frame hemisphere.
- **IMU (6-axis or 9-axis):** Provides node orientation for detection frame transform to torso frame. Also enables arm-gesture detection.
- **Short-range LiDAR / ToF (Optional):** Close-range geometry where radar resolution insufficient.

### Output

- **Haptic Actuator:** LRA or ERM for directional haptic alerts. Haptic encoder generates patterns encoding alert class and urgency.

### Communication

- **BAN Radio:** BLE 5.x or UWB. Transmits processed detection metadata (not raw sensor data). Receives haptic commands from belt controller.

### Connectivity-Agnostic Design

- **Local-Only:** Can operate entirely offline, transmitting only to the belt controller.
- **Cloud-Connected:** Can relay metadata via belt controller to cloud endpoint.
- **No data limits imposed:** Raw radar IQ, IMU at 1 kHz — if research requires it, the hardware design accommodates it.

---

## 5. Bracelet-Specific Design Notes

**Form factor:** PCB should be placed in the dorsal (back-of-hand) facing section of the band. Sensor apertures face away from wrist. Must not impede wrist flexion.

**Antenna design:** mmWave antennas at 60 GHz are integrable into the PCB silkscreen area. Antenna patch must face outward (away from wrist) to avoid body absorption.

**Water resistance:** IP67 mandatory. Sweat, rain, hand washing are guaranteed exposure conditions.

**Material contact:** All skin-contact surfaces must use hypoallergenic materials: titanium, surgical steel, medical-grade silicone, or hard-coat anodized aluminum (EN 1811 nickel release compliance required).

---

## 6. Interface Summary

| Interface | Signal Names | Connection |
|---|---|---|
| **mmWave Radar** | `SPI_MOSI/MISO/CLK`, `RADAR_CS`, `RADAR_IRQ` | SPI + GPIO |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI (shared bus, separate CS) |
| **Haptic** | `HAPTIC_SDA`, `HAPTIC_SCL` | I2C |
| **BAN Radio** | Integrated in MCU | — |
| **Charging** | `CHRG_IN+`, `CHRG_IN-` | Pogo contacts |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |

---

## 7. Directory Contents

```
bracelet_node/
├── bracelet_node.kicad_pro
├── bracelet_node.kicad_sch
├── bracelet_node.kicad_pcb
├── bracelet_node.kicad_prl
├── fp-info-cache
├── fp-lib-table
├── sym-lib-table
├── README.md
└── gerbers/
    ├── *.gbr
    └── *.drl
```

---

## 8. Related Documentation

- **Firmware:** `firmware/src/bin/bracelet_left.rs` / `bracelet_right.rs`
- **Sensing Pipeline:** `crates/sentinel-perception/src/radar_pipeline.rs`, `imu_pipeline.rs`
- **Haptic Logic:** `crates/sentinel-alerts/src/haptic_encoder.rs`
- **System Architecture:** `docs/theory/body_coordinate_fusion.md`
