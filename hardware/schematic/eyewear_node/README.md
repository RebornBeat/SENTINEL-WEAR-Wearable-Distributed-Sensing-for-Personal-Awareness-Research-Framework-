# Eyewear Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Face / glasses frame / headband
**Primary Role:** Forward-hemisphere fast-event detection, head-pose estimation, optional visual capture, SLAM visual odometry anchor
**Status:** Optional Node — Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture

**The eyewear node has NO WiFi and NO cellular connectivity.**

All eyewear data transmits via Body-Area Network (BAN) to the belt node exclusively. The belt node is the sole external network gateway.

```
Eyewear Sensors
       │
       │  BAN (BLE 5.x / UWB)
       ▼
  Belt Node (Fusion + Processing + External Gateway)
       │
       ├──► WiFi → Companion App (local)
       └──► Cellular → Companion App (remote)
```

**Data flow for live streaming:**
- Eyewear camera(s) encode video on-MCU or vision processor
- Encoded stream transmitted via BLE/UWB to belt node
- Belt node relays via WiFi or cellular to companion app
- Eyewear never connects to WiFi or external networks directly

**This is an architectural constraint enforced at hardware and firmware levels.**

---

## 2. Purpose

The Eyewear Node provides head-stabilized forward-hemisphere sensing. Because the head follows the wearer's gaze, this node always faces the direction of attention — the most valuable sensing direction for fast transient detection.

This node is **optional**. The system operates fully without it. It adds:
- Microsecond-latency fast object detection (event camera — ideal position)
- Head-pose estimation (separates head rotation from torso rotation in body-frame fusion)
- Optional conventional camera for forward visual capture
- Optional full camera array for SLAM visual odometry anchor
- Extreme velocity detection capability (event camera for fast transients)

---

## 3. Node Variants

### Variant A — Event-Only (Clip-On)

**Purpose:** Validated research platform. Clip-on form factor tests eyewear-position sensor value before committing to frame integration.

**Sensors:**
- Event-based camera (forward-facing)
- IMU (6-axis — head-pose estimation)

**Form factor:** Single PCB clip-on to nose bridge or frame. ~30 mm × 20 mm, ~5 g target.

**Use case:** 
- Determine sensing value of eyewear position
- Validate sensor selection before miniaturization
- Extreme velocity detection (event camera is optimal for this purpose)
- Research deployment with minimal complexity

**Battery:** 50–80 mAh. Expected runtime: 8–16 hours.

### Variant B — Event + Camera (Clip-On or Light Frame)

**Purpose:** Event camera for fast transient detection PLUS conventional camera for visual capture and SLAM contribution.

**Sensors:**
- Event-based camera (forward-facing, fast transient)
- Conventional camera (forward-facing, for visual recording, SLAM contribution)
- IMU (head-pose)

**Form factor:** Clip-on or integrated into lightweight glasses arm. ~10 g target.

**Use case:**
- Forward visual recording
- SLAM visual odometry contribution
- Combined fast detection + visual context
- The conventional camera provides loop closure for SLAM; the event camera provides fast-transient detection that conventional cameras miss
- Extreme velocity detection with visual verification

**Battery:** 50–80 mAh. Expected runtime: 4–8 hours.

### Variant C — Full Array (Frame-Integrated or Headband)

**Purpose:** Multi-camera array covering the forward 180°+ field from the head position. Maximum SLAM and visual coverage contribution.

**Sensors:**
- Event camera (forward-facing)
- Conventional camera (forward-facing, primary visual)
- Side cameras (left and right temporal, ~90° from forward)
- IMU (head-pose, high rate ≥ 200 Hz)

**Form factor options:**

| Option | Description | Advantages | Disadvantages |
|--------|-------------|------------|---------------|
| **Frame-integrated** | Cameras in frame arms and bridge | Aesthetic, always-worn | PCB extremely constrained, custom frames required |
| **Headband** | Elastic headband with forehead PCB | Easier to prototype, more space | Less aesthetic, separate accessory |
| **Cap-integrated** | PCB in cap brim | Ample space, solar option | Requires cap, weather dependent |

**Purpose:** This variant is the SLAM visual odometry anchor. Forward + side camera coverage provides the baseline for continuous 3D reconstruction as the wearer moves. Combined with pendant 360° cameras and anklet ToF, provides complete environmental capture.

**Battery:** 80–150 mAh (integrated) or belt power cable (research mode).

**Expected runtime (integrated battery):** 2–4 hours (camera-heavy).

**With belt power cable:** Unlimited runtime (drawing from belt battery).

**Use case:** Maximum SLAM fidelity. Users running dense world model mode. Professional deployments.

### Variant D — Event-Enhanced (Combined Event + Conventional Array)

**Purpose:** Maximum detection capability including extreme velocity detection plus full visual coverage.

**Sensors:**
- 2 event cameras (forward + one side position)
- 2-4 conventional cameras (forward + sides)
- IMU (head-pose)

**Use case:** Users requiring the full sensing stack including Doppler-triggered fast transient capture. Event cameras provide microsecond detection; conventional cameras provide visual context and SLAM data.

**Form factor:** Headband recommended for research phase.

---

## 4. Candidate Component Set (All Variants — Not Locked)

### MCU

| Variant | Recommended MCU | Rationale |
|---------|-----------------|-----------|
| A | Nordic nRF5340 | Consistent with other nodes, firmware uniformity, BLE 5.3 integrated |
| B | STM32L4+ | Ultra-low power, camera interface (DVP/CSI) available |
| C | NXP i.MX RT1062 | Cortex-M7, 600 MHz, multiple camera interfaces, handles multiplexed streams |
| D | NXP i.MX RT1062 or dedicated vision processor | Multiple camera streams + event camera interface |

**Architecture note:** For Variants C and D with multiple cameras, compute offload to belt node is the recommended architecture. Eyewear node performs:
- Camera multiplexing / selection
- On-node encoding (H.264/H.265)
- IMU fusion
- Event camera processing
- Transmission to belt via BLE/UWB

Belt node handles:
- SLAM processing
- Multi-camera stitching (if applicable)
- Recording management
- Relaying to companion app

### Event Camera

| Module | Resolution | Interface | Power | Notes |
|--------|------------|-----------|-------|-------|
| Prophesee Metavision EVK3 | 640×480 | USB | ~200 mW | Evaluation, larger form factor |
| iniVation DAVIS346 | 346×260 | USB | ~150 mW | Combined frame + events |
| Sony IMX636 | 1280×720 | MIPI CSI-2 | ~100 mW | Compact, production-grade |
| Custom ASIC | Variable | SPI/parallel | < 50 mW | Research target for frame integration |

**Selection criteria:**
- Physical size (PCB ≤ 15 mm × 10 mm for frame-integrated variants)
- Power consumption (glasses battery is severely constrained)
- Resolution sufficient for fast-object detection at body-frame distances (1–5 m)
- Latency (must be < 1 ms for extreme velocity detection)

**For extreme velocity detection:** Event camera must have microsecond latency. Most commercial event cameras satisfy this requirement.

### Conventional Camera (Variants B, C, D)

| Module | Resolution | Interface | Power | Notes |
|--------|------------|-----------|-------|-------|
| OV2640 | 2 MP | DVP / SPI | ~80 mW | Compact, low power |
| OV5640 | 5 MP | MIPI CSI-2 | ~150 mW | Auto-focus option |
| IMX219 | 8 MP | MIPI CSI-2 | ~200 mW | High quality |
| IMX477 | 12 MP | MIPI CSI-2 | ~400 mW | Premium quality, higher power |
| OV7251 | VGA | MIPI CSI-2 | ~50 mW | Global shutter, fast capture |
| HM01B0 | QVGA | SPI | ~2 mW | Ultra-low power, on-chip detection |

**For Variant C/D side cameras:** OV2640 or equivalent with wide-angle or fisheye lens.

**Data handling:** Fully user-configured per `sentinel-wear.toml`. Options:
- Metadata-only: On-device inference, transmit classification only
- Local storage: Frames to SD card or flash
- App streaming: Encoded stream to belt → WiFi → companion app
- Continuous recording: All frames captured with integrity chain

### IMU

| Module | Noise Density | Interface | Package | Notes |
|--------|---------------|-----------|---------|-------|
| Bosch BMI270 | 80 µg/√Hz | SPI/I²C | 2.5 × 2.5 mm | Consistent with all nodes |
| TDK ICM-42688-P | 70 µg/√Hz | SPI/I²C | 2.0 × 1.7 mm | Smallest package — critical for glasses |
| ST LSM6DSV | 60 µg/√Hz | SPI/I²C | 2.0 × 1.7 mm | Built-in ML for gesture |

**IMU role in eyewear:**
- Head-pose estimation independent of torso
- Head-torso separation for body-frame fusion
- Distinguishes "wearer looked left" from "wearer turned left"
- Required for visual stabilization during SLAM
- Rate: ≥ 200 Hz for stable head tracking

### BAN Radio

| Radio | Bandwidth | Power | Notes |
|-------|-----------|-------|-------|
| BLE 5.3 (MCU integrated) | ~500 Kbps sustained | 5-15 mW | Primary transport, always present |
| UWB (DW3000 class) | 4-6 Mbps sustained | 50-150 mW | Optional for camera streaming |

**Configuration:**
```toml
[nodes.eyewear.ban]
primary = "ble"
uwb_enabled = false          # Enable if camera streaming over UWB
uwb_role = "full_bandwidth"  # For video streaming
```

### Battery

| Variant | Capacity | Chemistry | Expected Runtime |
|---------|----------|-----------|------------------|
| A | 50–80 mAh | LiPo | 8–16 hours (event-only) |
| B | 50–80 mAh | LiPo | 4–8 hours (event + camera) |
| C | 80–150 mAh | LiPo | 2–4 hours (full array) |
| D | 80–150 mAh | LiPo | 2–4 hours (full array) |

**Belt power option (Variants C, D):**
- Thin cable from belt node (through shirt, behind neck)
- Supplies 5V from belt battery
- Eliminates eyewear battery constraint for research deployments
- Useful for extended SLAM sessions, security details

### Charging

| Method | Suitable For | Notes |
|--------|--------------|-------|
| USB-C (micro) | Clip-on variants | Smallest USB connector |
| Magnetic pogo pins | All variants | Docking station, eyewear-friendly |
| Qi wireless | Headband variant | Larger space for coil |
| Contact charging | Frame-integrated | Magnetic connector on temple |

---

## 5. Extreme Velocity Detection Integration

### 5.1 Why Eyewear is Optimal for Extreme Velocity Detection

The eyewear position is the best location for fast transient detection because:

| Factor | Benefit |
|--------|---------|
| Head follows gaze | Camera faces where wearer is looking |
| Minimal body occlusion | Forward hemisphere is unobstructed |
| Event camera latency | < 1 ms detection possible |
| Head is stable reference | During threat detection, head naturally tracks threat |

### 5.2 Reaction Time Budget

At 3-meter engagement distance:
- Rifle velocity (850 m/s): **3.5 ms** from entry to impact
- Handgun velocity (350 m/s): **8.5 ms** from entry to impact

**Detection budget breakdown:**

| Stage | Latency | Notes |
|-------|---------|-------|
| Event camera detection | < 100 µs | Hardware event generation |
| MCU interrupt | < 10 µs | Real-time priority |
| Classification | 100–500 µs | Fast object detection |
| BLE queue (with QoS Critical) | < 2 ms | Preempts other traffic |
| BLE transmission | 1–2 ms | Packet airtime |
| Belt reception | < 500 µs | Processing |
| Alert trigger | < 500 µs | Haptic/audio |
| **Total** | **~4–6 ms** | Within rifle detection window |

**With QoS Critical enabled:** Detection message preempts all other BLE traffic, ensuring reliable low-latency delivery.

### 5.3 Configuration for Extreme Velocity Detection

```toml
[extreme_velocity]
enabled = true

[extreme_velocity.eyewear]
event_camera_always_on = true
detection_threshold_ms = 0.1      # 100 µs threshold
qos_priority = "critical"         # Preempts other BLE traffic
min_velocity_ms = 50              # Ignore objects below this
max_detection_distance_m = 10     # Relevant detection range
trigger_forward_event_camera = true
trigger_side_event_camera = false # Only forward for initial version
```

### 5.4 Event Camera Requirements for Extreme Velocity

| Requirement | Value |
|-------------|-------|
| Latency | < 1 ms to detection event |
| Resolution | QVGA (320×240) minimum, higher not required |
| Field of view | ≥ 90° horizontal for forward hemisphere |
| Dynamic range | High (outdoor / indoor transitions) |
| Update rate | Event-based (no fixed frame rate) |

---

## 6. SLAM Contribution

### 6.1 Variant A (Event-Only) — Limited SLAM

**Contribution:** Sparse optical flow from event camera. Fast-feature tracking between frames. No dense reconstruction.

**Use case:** Body-frame motion estimation, heading changes, fast-motion tracking.

**Limitation:** No keyframe images for loop closure. Drift accumulates without visual reference images.

### 6.2 Variant B (Event + Camera) — Strong SLAM

**Contribution:**
- Conventional camera: keyframes for loop closure
- Event camera: smooth inter-frame tracking
- IMU: visual-inertial odometry

**Result:** High-quality visual odometry from the head position. The head follows gaze, so SLAM tracks where the wearer is looking.

**Use case:** Dense SLAM with visual reference images. Loop closure when revisiting locations.

### 6.3 Variant C (Full Array) — Maximum SLAM

**Contribution:**
- Forward camera: primary visual odometry
- Side cameras: depth from stereo (wide baseline between forward and side cameras)
- Event camera: fast-motion interpolation
- IMU: visual-inertial fusion

**Result:** Complete 180°+ visual coverage from head position. Prevents visual odometry failure during rapid head rotation. This is the visual anchor for dense world model.

**Stereo depth capability:** Forward camera + left side camera provides wide-baseline stereo. Forward camera + right side camera provides second stereo pair. Combined: robust depth estimation without dedicated depth sensor.

### 6.4 Integration with Pendant 360° and Anklet ToF

```
Eyewear Node (Variant C)
    │
    ├── Forward/side cameras → Visual odometry, SLAM keyframes
    ├── Event camera → Fast transient detection, motion interpolation
    └── IMU → Head-pose
            │
            ▼
    Combined with:
    │
    ├── Pendant 360° cameras → Full body-level 360° visual
    ├── Pendant IMU → Torso reference
    ├── Anklet ToF → Ground plane, floor obstacles
    └── Belt IMU → Body-frame origin
            │
            ▼
    Dense World Model (at belt node)
```

---

## 7. Head-Torso Separation

### 7.1 The Problem

Without eyewear node IMU, body-frame fusion assumes head and torso are aligned. In reality:
- Head turns left/right while torso faces forward
- Head tilts up/down while walking
- Head moves independently during conversation

This causes detection attribution errors: a detection from "left of head" might actually be "forward of torso" if the wearer looked left.

### 7.2 The Solution

With eyewear IMU, the system computes:
- Head orientation (eyewear IMU)
- Torso orientation (belt IMU)
- Head-torso extrinsic: the transform between them

```
Body-Frame Fusion:
    Torso frame (+Y forward from belt IMU)
         │
         └── Head-torso extrinsic (computed from IMU difference)
                 │
                 └── Head frame (+Y forward from eyewear IMU)
                         │
                         └── Detections from eyewear cameras
                                 │
                                 └── Transformed to torso frame for fusion
```

### 7.3 Configuration

```toml
[nodes.eyewear.body_frame]
head_torso_separation = true
imu_rate_hz = 200
extrinsic_calibration_mode = "auto"  # "auto" | "manual"
```

---

## 8. Form Factor Design Notes

### 8.1 Clip-On Form (Variants A, B)

**Dimensions:** ~30 mm × 20 mm × 5 mm

**Weight:** 5–10 g (with battery)

**Attachment:**
- Nose bridge clip (for glasses)
- Frame front clip (for glasses top bar)
- Rubberized grip for secure attachment

**Design considerations:**
- Center of gravity must be balanced
- Wire routing to charging contacts
- LED indicator for status
- Debug pads accessible from side

**Materials:**
- PCB: Standard FR4 or flexible PCB
- Enclosure: 3D printed PETG or injection molded polycarbonate
- Clip: Silicone over-mold or metal spring

### 8.2 Frame-Integrated (Variant C)

**The constraint:** A standard glasses temple arm is approximately:
- Width: 5 mm
- Thickness: 2 mm
- Length: 70–80 mm

**PCB dimensions that fit:**
- Per arm: 60 mm × 3 mm × 1 mm (extremely constrained)
- Bridge area: 10–15 mm height, more space available

**Component height constraint:**
- Surface-mount components must be < 0.5 mm height for slim sections
- Camera modules require at least 3 mm height
- Batteries require at least 2 mm height

**Recommended approach for v1:** Do not attempt frame integration until clip-on validation is complete. Frame integration is a Phase 5+ engineering challenge requiring:
- Custom ultra-thin PCB stack
- Specialized camera modules
- Flex PCB interconnects between arms and bridge
- Custom frame manufacturing

### 8.3 Headband (Variant C Alternative)

**Dimensions:** 
- Central forehead PCB: 30–40 mm × 30–40 mm × 8–10 mm
- Flexible extensions: 2–3 mm thick, to temples

**Advantages:**
- Ample space for components
- Comfortable for extended wear
- Easy to prototype
- Accessible for debugging
- Suitable for users without glasses

**Disadvantages:**
- Less aesthetic
- Separate accessory to remember
- May interfere with hats

**Materials:**
- Central PCB: Standard FR4
- Flexible extensions: Polyimide flex PCB
- Enclosure: 3D printed or silicone over-mold
- Band: Elastic or adjustable fabric

### 8.4 Cap-Integrated (Alternative to Headband)

**Dimensions:**
- PCB in cap brim: 60–80 mm × 15–20 mm × 5 mm

**Advantages:**
- Ample space
- Natural sun shading
- Optional solar charging (small panel on brim top)
- Fashion-neutral (caps are common)

**Disadvantages:**
- Requires cap
- Weather dependent
- Not suitable for formal occasions

---

## 9. Power Architecture

### 9.1 Power Budget

| Component | Active Power | Notes |
|-----------|--------------|-------|
| MCU (sleep) | 0.5–2 mW | Deep sleep mode |
| MCU (active) | 10–30 mW | Processing |
| Event camera | 100–200 mW | Continuous operation |
| Conventional camera | 80–200 mW | Per camera |
| BLE radio (TX) | 20–50 mW | During transmission |
| UWB radio (TX) | 50–150 mW | If enabled |
| IMU | 1–5 mW | Continuous |

### 9.2 Total Power by Variant

| Variant | Configuration | Total Power | Runtime (80 mAh) |
|---------|---------------|-------------|-------------------|
| A (event-only) | Event cam + IMU + BLE idle | 150–250 mW | 12–16 hours |
| B (event + camera) | All above + 1 camera | 250–450 mW | 4–8 hours |
| C (full array) | Event + 3–4 cameras + IMU | 400–800 mW | 2–4 hours |

### 9.3 Power Management

**Duty cycling strategies:**

| Strategy | Power Reduction | Impact |
|----------|-----------------|--------|
| Camera FPS reduction (30→15 fps) | 30–40% | Minor quality impact |
| Event camera only during idle | 60–80% | No visual SLAM contribution |
| BLE advertising only (no streaming) | 10–20% | Detection still works |
| Deep sleep between detection cycles | 90%+ | Latency increases |

**Configuration:**

```toml
[nodes.eyewear.power]
mode = "balanced"  # "performance" | "balanced" | "power_saving"
camera_fps = 15
event_always_on = true
ble_interval_ms = 30
deep_sleep_after_idle_minutes = 5
```

### 9.4 Belt Power Input (Extended Runtime)

For research deployments where a cable is acceptable:

| Signal | Description |
|--------|-------------|
| `V_BELT_IN` | 5V from belt battery |
| `V_BELT_GND` | Return ground |
| `V_BELT_EN` | Enable signal (belt controls when to supply) |

**Physical implementation:**
- Thin flexible cable (2–3 mm diameter)
- Routes from belt, under shirt, behind neck, to eyewear
- Breakaway connector for safety
- Magnetic attachment option

---

## 10. Thermal Considerations

### 10.1 Thermal Constraints

**Maximum temperature at skin contact:** 42°C

**Thermal challenges:**
- Event camera generates heat continuously
- Multiple cameras generate significant heat
- Small enclosure has limited thermal mass
- Direct skin contact (ears, nose bridge, forehead)

### 10.2 Thermal Mitigation

| Strategy | Effectiveness | Notes |
|----------|---------------|-------|
| Thermal vias under ICs | Moderate | Spread heat to enclosure |
| Air gap between PCB and enclosure | Moderate | Convection |
| Enclosure material (metal) | High | Heat spreading |
| Reduced FPS / duty cycling | High | Lower power = less heat |
| Clip-on vs frame-integrated | Moderate | Clip-on has more surface area |

### 10.3 Thermal Monitoring

```toml
[nodes.eyewear.thermal]
monitoring_enabled = true
thermistor_pin = "NTC1"
warning_threshold_c = 38
throttle_threshold_c = 40
shutdown_threshold_c = 45
throttle_action = "reduce_fps"
```

---

## 11. Camera Data Handling

All camera data handling is user-configured. No system-imposed restrictions.

### 11.1 Data Handling Modes

| Mode | Description | Bandwidth | Storage |
|------|-------------|-----------|---------|
| Metadata-only | On-device inference, classification tags only | ~10 KB/s | None |
| Local storage | Frames to SD/flash | Variable | SD/flash |
| App streaming | Encoded stream to belt → app | 1–4 Mbps | None |
| Continuous recording | All frames captured | 2–8 Mbps | SD/flash |

### 11.2 Configuration

```toml
[nodes.eyewear.data]
mode = "metadata_only"  # "metadata_only" | "local_storage" | "app_streaming" | "continuous"
store_raw = false
stream_quality = "medium"  # "low" | "medium" | "high" | "raw"
recording_trigger = "on_detection"
```

### 11.3 Optional Hardware Power Switch

If user wants unconditional physical camera disable:
- Hardware switch cuts `V_CAM` rail
- PCB provides footprint + MOSFET
- When disabled: camera physically unpowered, no software override

---

## 12. Antenna and RF Considerations

### 12.1 BLE Antenna

**Single antenna (standard):**
- Integrated PCB antenna or chip antenna
- Omnidirectional pattern
- Body shadowing affects link

**Antenna diversity (optional):**
- Two antennas (left/right side of eyewear)
- Antenna switch selects best signal
- Reduces body shadowing impact
- Improves link reliability

**Configuration:**

```toml
[nodes.eyewear.antenna_diversity]
enabled = false  # Enable if dual-antenna PCB populated
selection_mode = "rssi"
```

### 12.2 RF Coexistence

| Radio | Frequency | Conflict Risk |
|-------|-----------|---------------|
| BLE | 2.4 GHz | WiFi at belt |
| UWB | 3.1–10.6 GHz | None |
| Event camera | Optical | None |

**Mitigation:** Eyewear is physically separated from belt WiFi antenna (on torso). No direct coexistence issues.

---

## 13. Interface Summary

### 13.1 Sensor Interfaces

| Interface | Signal Names | Connection |
|-----------|--------------|------------|
| Event Camera | USB D+/D- or SPI | USB 2.0 or SPI parallel |
| Conventional Camera(s) | `MIPI_CLK+/-`, `MIPI_D0+/-` | MIPI CSI-2 per camera |
| IMU | `IMU_CS`, `IMU_INT`, `SPI_CLK/MOSI/MISO` | SPI |
| Environmental (optional) | `I2C_SDA/SCL` | I²C shared bus |

### 13.2 BAN Radio

| Interface | Signal Names | Notes |
|-----------|--------------|-------|
| BLE | Integrated in MCU | Primary transport |
| UWB (optional) | `UWB_CS`, `UWB_IRQ`, `SPI_*` | DW3000 via SPI |

### 13.3 Power and Debug

| Interface | Signal Names | Notes |
|-----------|--------------|-------|
| Battery | `VBAT+`, `VBAT-` | LiPo |
| Charging | `CHRG_IN+`, `CHRG_IN-` | USB or pogo pins |
| Belt Power (optional) | `V_BELT_IN`, `V_BELT_GND` | 5V from belt |
| Debug | `SWD_CLK`, `SWD_DATA`, `UART_TX` | 4-pin header |

### 13.4 Test Pads

| Pad | Signal | Description |
|-----|--------|-------------|
| 1 | `VCC_BATT` | Battery voltage |
| 2 | `GND` | Ground |
| 3 | `CHRG_IN+` | Charging input |
| 4 | `CHRG_IN-` | Charging return |
| 5 | `BAN_RF_TEST` | BLE antenna tap |
| 6 | `MCU_BOOT` | Boot mode select |
| 7 | `MCU_RESET` | MCU reset |
| 8 | `CAM_HW_SW_TEST` | Camera switch state (if installed) |
| 9 | `SWD_CLK` | SWD clock |
| 10 | `SWD_DATA` | SWD data |
| 11 | `UART_TX` | Debug console |
| 12 | `UART_RX` | Debug console |

---

## 14. Mechanical Specifications

### 14.1 Clip-On Dimensions

| Variant | Length | Width | Height | Weight |
|---------|--------|-------|--------|--------|
| A (event-only) | 30 mm | 20 mm | 5 mm | 5–8 g |
| B (event + camera) | 35 mm | 22 mm | 6 mm | 8–12 g |

### 14.2 Headband Dimensions

| Component | Dimensions |
|-----------|------------|
| Central forehead PCB | 30–40 mm × 30–40 mm × 8–10 mm |
| Flexible extensions | 2–3 mm thick, 50–70 mm length |
| Total weight | 20–40 g (depending on battery) |

### 14.3 Frame-Integrated Dimensions

| Component | Dimensions |
|-----------|------------|
| Temple arm PCB (each) | 60 mm × 3 mm × 1 mm |
| Bridge PCB | 30 mm × 15 mm × 5 mm |
| Total weight | 15–25 g (with battery in temple) |

---

## 15. Testing Requirements

### 15.1 All Variants

1. Power-on self-test
2. Event camera initialization and response test
3. IMU enumeration and gravity vector verification
4. BLE connectivity to belt node (ping latency < 20 ms)
5. Battery level reporting

### 15.2 Variants B, C, D (With Camera)

1. Camera initialization
2. Frame capture test
3. Data handling mode verification:
   - Metadata-only: classification tag transmitted, no storage
   - Local storage: file written within 500 ms
   - App streaming: belt receives stream within 2 seconds
4. Optional hardware switch test (if installed): V_CAM = 0V when disabled

### 15.3 Extreme Velocity Detection Test

1. Event camera latency test: LED strobe response < 1 ms
2. QoS Critical transmission test: Detection message arrives at belt within 5 ms
3. End-to-end alert test: Strobe → Detection → Alert within 10 ms

---

## 16. Directory Structure

```
eyewear_node/
├── variant_a_event_only/
│   ├── eyewear_event.kicad_pro
│   ├── bom/
│   └── gerbers/
├── variant_b_event_camera/
│   ├── eyewear_event_cam.kicad_pro
│   ├── bom/
│   └── gerbers/
├── variant_c_full_array/
│   ├── eyewear_array_main.kicad_pro
│   ├── eyewear_array_headband.kicad_pro
│   ├── bom/
│   └── gerbers/
├── variant_d_event_enhanced/
│   ├── eyewear_event_enhanced.kicad_pro
│   ├── bom/
│   └── gerbers/
├── bom/
│   └── shared_components.csv
└── README.md
```

---

## 17. Configuration Reference

```toml
# Eyewear Node Configuration

[nodes.eyewear]
enabled = false                    # Opt-in node
variant = "b"                      # "a" | "b" | "c" | "d"
sensors = ["EventCamera", "Camera", "Imu"]
form_factor = "clip_on"            # "clip_on" | "headband" | "frame_integrated"

[nodes.eyewear.event_camera]
always_on = true
latency_threshold_ms = 0.1
resolution = "vga"                 # "qvga" | "vga" | "svga"

[nodes.eyewear.conventional_camera]
enabled = true
count = 1                          # 1 | 2 | 3 | 4
resolution = "2mp"                 # "qvga" | "vga" | "2mp" | "5mp" | "8mp"
fps = 15
data_mode = "metadata_only"        # "metadata_only" | "local_storage" | "app_streaming" | "continuous"

[nodes.eyewear.imu]
rate_hz = 200
head_torso_separation = true

[nodes.eyewear.ban]
primary = "ble"
uwb_enabled = false
uwb_role = "full_bandwidth"

[nodes.eyewear.antenna_diversity]
enabled = false

[nodes.eyewear.power]
mode = "balanced"
deep_sleep_after_idle_minutes = 5

[nodes.eyewear.thermal]
monitoring_enabled = true
warning_threshold_c = 38
throttle_threshold_c = 40

[nodes.eyewear.extreme_velocity]
enabled = false
event_camera_trigger = true
qos_priority = "critical"
min_velocity_ms = 50
max_detection_distance_m = 10
```
