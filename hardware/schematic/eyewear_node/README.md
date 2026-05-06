# Eyewear Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Face / glasses frame / headband
**Primary Role:** Forward-hemisphere fast-event detection, head-pose estimation, optional visual capture
**Status:** Optional Node. Reference Design — All Variants.
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Eyewear Node provides head-stabilized forward-hemisphere sensing. Because the head follows the wearer's gaze, this node always faces the direction of attention — the most valuable sensing direction.

This node is optional. The system operates fully without it. It adds:
- Microsecond-latency fast object detection (event camera — the ideal position for one)
- Head-pose estimation (separates head rotation from torso rotation in body-frame fusion)
- Optional conventional camera for forward visual capture
- Optional full camera array for SLAM visual odometry anchor

---

## 2. Node Variants

### Variant A — Event-Only (Clip-On)

**Purpose:** Validated research platform. Clip-on form factor tests eyewear-position sensor value before committing to frame integration.

**Sensors:**
- Event-based camera (forward-facing)
- IMU (6-axis — head-pose estimation)

**Form factor:** Single PCB clip-on to nose bridge or frame. ~30 mm × 20 mm, ~5 g target.

**Use case:** Determine what sensing value the eyewear position adds. Validate sensor selection before miniaturization effort.

### Variant B — Event + Camera (Clip-On or Light Frame)

**Purpose:** Event camera for fast transient detection PLUS conventional camera for visual capture.

**Sensors:**
- Event-based camera (forward-facing, fast transient)
- Conventional camera (forward-facing, for visual recording, SLAM contribution)
- IMU (head-pose)

**Form factor:** Clip-on or integrated into lightweight glasses arm. ~10 g target.

**Use case:** Forward visual recording. SLAM visual odometry contribution. The conventional camera provides loop closure for SLAM; the event camera provides fast-transient detection that conventional cameras miss.

### Variant C — Full Array (Frame-Integrated or Headband)

**Purpose:** Multi-camera array covering the forward 180°+ field from the head position. Maximum SLAM and visual coverage contribution.

**Sensors:**
- Event camera (forward-facing)
- Conventional camera (forward-facing)
- Side cameras (left and right temporal, ~90° from forward)
- IMU (head-pose, high rate)

**Form factor options:**
- **Option A:** Glasses frame integrated — cameras in frame arms and bridge
- **Option B:** Headband — more space, easier to prototype, for non-glasses users

**Purpose:** This variant is the SLAM visual odometry anchor. Forward + side camera coverage provides the baseline for continuous 3D reconstruction as the wearer moves. Combined with pendant 360° cameras and anklet ToF, provides complete environmental capture.

**Use case:** Maximum SLAM fidelity. Users running dense world model mode.

---

## 3. Candidate Component Set (All Variants — Not Locked)

### MCU

- **Variant A:** Nordic nRF5340 (consistent with other nodes — firmware uniformity)
- **Variant B:** STM32L4+ (ultra-low power, camera interface available)
- **Variant C:** NXP i.MX RT1062 (Cortex-M7, 600 MHz, handles multiple camera streams + IMU fusion)

For Variant C (multiple cameras), the MCU must handle multiple CSI-2 interfaces or a camera multiplexer. Compute offload to belt node is the recommended architecture — eyewear node compresses and transmits; belt node processes.

### Event Camera

- **Variant A:** Prophesee Metavision EVK3 (640×480, USB, lower power) — evaluation
- **Variant B:** iniVation DAVIS346 (346×260, combined frame+events, USB) — combined output
- **Variant C:** Custom ASIC from Prophesee or iniVation at reduced resolution for wearable PCB integration — research target
- **Variant D:** Sony IMX636-based compact module

**Selection criteria:**
- Physical size (PCB ≤ 15 mm × 10 mm for frame-integrated variants)
- Power consumption (glasses battery is severely constrained — 50–80 mAh total)
- Resolution sufficient for fast-object detection at body-frame distances (1–5 m)

### Conventional Camera (Variants B, C)

- **Forward-facing:** OmniVision OV2640 (2 MP, DVP/SPI), or Sony IMX219 (8 MP, CSI-2)
- **Side cameras (Variant C):** OmniVision OV2640 with wide-angle or fisheye lens per position

**Camera data handling:** Fully user-configured per `sentinel-wear.toml`. No system-imposed restrictions. Optional hardware power switch if user installs one.

### IMU

- **Variant A:** Bosch BMI270 (consistent with all nodes)
- **Variant B:** TDK ICM-42688-P (smallest package — critical for glasses)

### BAN Radio

- BLE 5.3 via MCU integrated radio (primary)

### Battery

- **Variant A, B:** LiPo 3.7V, 50–80 mAh — the most constrained node
- **Variant C:** LiPo 3.7V, 80–150 mAh (slightly larger for multi-camera draw)
- Expected runtime:
  - Variant A event-only: 8–16 hours
  - Variant B event+camera: 4–8 hours
  - Variant C full array: 2–4 hours (camera-heavy variant; may supplement from belt via cable)

**Belt power option (Variant C):** A thin cable from the belt node supplies power through the glasses/headband. Eliminates glasses battery for research deployments where cable is acceptable.

---

## 4. Form Factor Design Notes

### Clip-On Form (Variants A, B)

Single PCB, approximately 30 mm × 20 mm × 5 mm. Clips to nose bridge or frame front. Weight: 5–10 g. Tests eyewear-position sensor value before frame integration work.

### Frame-Integrated (Variant C)

**The constraint:** A standard glasses temple arm is approximately 5 mm wide × 2 mm thick × 70 mm long. PCBs that fit within both arms: approximately 60 mm × 3 mm × 1 mm per arm. This is extremely challenging — surface-mount components must be < 0.5 mm height for the slim sections.

**Recommended approach for v1:** Do not attempt frame integration until clip-on validation is complete. Frame integration is a Phase 5+ engineering challenge.

**Bridge section:** The frame front (bridge + lens area) provides more space — 10–15 mm vertical height. Event camera, forward conventional camera, and IMU can fit here. Side cameras route to arm PCBs.

### Headband (Variant C Alternative)

Elastic headband with a 30 mm × 30 mm central PCB at the forehead position. Substantially easier to prototype. Camera modules on flexible extensions at temporal positions. Suitable for users without glasses and for all research phases before miniaturization.

---

## 5. SLAM Contribution

**Variant A (event-only):** Limited SLAM contribution. Event camera provides fast-feature sparse optical flow but no keyframe images for dense reconstruction.

**Variant B (event + camera):** Strong SLAM contribution. Conventional camera provides keyframes for loop closure. Event camera provides smooth inter-frame tracking. Together: high-quality visual odometry from the head position.

**Variant C (full array):** Maximum SLAM contribution. Wide-baseline stereo from forward + side cameras provides depth from stereo without dedicated depth sensor. Full 180°+ coverage prevents visual odometry failure during rotation. This is the visual anchor for the dense world model.

---

## 6. Interface Summary

| Interface | Signal Names | Connection |
|---|---|---|
| **Event Camera** | USB or SPI parallel | USB 2.0 or SPI |
| **Conventional Camera(s)** | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 per camera |
| **IMU** | `IMU_CS`, `IMU_INT` | SPI |
| **BAN Radio** | Integrated in MCU | — |
| **Battery / Charging** | `CHRG_IN+`, `CHRG_IN-` | Pogo pins or USB |
| **Debug** | `SWD_CLK`, `SWD_DIO`, `UART_TX` | 4-pin header |
| **Optional Belt Power** | `V_BELT_IN` (5V) | Thin cable from belt |

---

## 7. Directory Structure

```
eyewear_node/
├── variant_a_event_only/
│   ├── eyewear_event.kicad_pro
│   └── gerbers/
├── variant_b_event_camera/
│   ├── eyewear_event_cam.kicad_pro
│   └── gerbers/
├── variant_c_full_array/
│   ├── eyewear_array_main.kicad_pro
│   ├── eyewear_array_headband.kicad_pro
│   └── gerbers/
├── bom/
└── README.md
```
