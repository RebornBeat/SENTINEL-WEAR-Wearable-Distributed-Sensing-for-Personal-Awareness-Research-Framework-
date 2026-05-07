# Bracelet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Wrist/forearm — worn on both left and right wrists
**Primary Role:** Forearm-hemisphere sensing, directional haptic output, optional visual capture
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Purpose

The Bracelet Node provides forearm-hemisphere sensing and the primary directional haptic alert output. Two nodes (left and right wrist) together cover lateral body hemispheres and discriminate forward/rear approaches.

All variants are camera-free by default. A camera module can be optionally populated for users wanting arm-direction visual capture.

---

## 2. Connectivity Architecture

**The bracelet node has NO WiFi and NO cellular connectivity.**

All bracelet data transmits via Body-Area Network (BAN) to the belt node exclusively. The belt node is the sole external network gateway.

```
Bracelet sensors → BAN (BLE 5.x / UWB) → Belt Node → WiFi/Cellular → Companion App
```

If the user has configured live video streaming, the video is encoded on the bracelet, transmitted via BAN to the belt node, and the belt node relays it over WiFi/cellular to the companion app.

**Data flow:**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BRACELET NODE DATA FLOW                              │
│                                                                              │
│  Radar/IMU/Optional Camera                                                   │
│         │                                                                    │
│         ▼                                                                    │
│  On-node processing (detection, compression)                                 │
│         │                                                                    │
│         ▼                                                                    │
│  BAN Radio (BLE 5.x primary / UWB optional)                                  │
│         │                                                                    │
│         ▼                                                                    │
│  Belt Node (hub)                                                             │
│         │                                                                    │
│         ├──► Local Storage (SD card)                                         │
│         └──► WiFi / Cellular → Companion App                                 │
│                                                                              │
│  ⚠️  BRACELET NODE = BAN ONLY (no external connectivity)                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Design Philosophy

No artificial limits on sensor selection, storage capacity, or data transmission. The architecture is defined by its interface standards, not its components.

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at maximum speeds. Resolution, storage, compute are configuration choices.
- **Modularity:** Each functional block defined by interface, not part number.
- **Connectivity Agnostic:** BLE 5.x primary BAN transport; UWB optional for bandwidth and precision.
- **User-Controlled Data Handling:** No system-imposed restrictions on storage, streaming, or retention.

**Privacy controls:** Any camera installed on a bracelet node follows user-configured data handling policy. No system-imposed restrictions. Optional hardware power switch available to users who want physical camera disable.

---

## 4. Hardware Architecture

### 4.1 Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      BRACELET NODE PCB                           │
├──────────────────────────────────────────────────────────────────┤
│  MCU Core                                                        │
│  - ARM Cortex-M33 (nRF5340) or Cortex-M4 (STM32WL55)            │
│  - FPU required for IMU fusion                                    │
│  - BLE 5.3 integrated radio                                       │
├──────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces                                               │
│  - mmWave Radar (SPI/UART)                                        │
│  - IMU (SPI/I2C)                                                  │
│  - Optional ToF (I2C)                                             │
│  - Optional Camera (MIPI CSI-2 / DVP)                             │
├──────────────────────────────────────────────────────────────────┤
│  BAN Radio                                                        │
│  - BLE 5.3 (mandatory, integrated)                                │
│  - UWB optional (DW3000 footprint)                                │
│  - NO WiFi, NO Cellular                                           │
├──────────────────────────────────────────────────────────────────┤
│  Storage Interface (Optional)                                      │
│  - microSD card (SPI, up to 2 TB)                                 │
│  - Internal QSPI NOR Flash                                        │
├──────────────────────────────────────────────────────────────────┤
│  User Interface                                                    │
│  - Haptic Driver (I2C DRV2605L) + LRA/ERM actuator               │
│  - Status LEDs (GPIO)                                             │
│  - Optional hardware camera power switch                          │
├──────────────────────────────────────────────────────────────────┤
│  Power Management                                                 │
│  - Battery Charger + Fuel Gauge (I2C)                            │
│  - Buck/Boost Regulators                                          │
├──────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                  │
│  - SWD (Serial Wire Debug)                                        │
│  - UART Console (115200 baud)                                     │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Single PCB Design — Configuration Options

**Important architectural decision:** The bracelet node uses a **single PCB design** that supports all variants through configuration options (component population choices) rather than separate hardware variants.

**This means:**
- One PCB layout supports minimal, standard, and extended configurations
- Variant differences are determined at assembly time (which components are populated)
- User can select sensor set, camera presence, UWB presence, and battery capacity

**Benefits:**
- Reduced SKU count
- Simplified manufacturing
- Easier inventory management
- Flexibility for field upgrades (if footprints are populated but components not)

---

## 5. Configuration Options

The following are **configuration options** (not separate hardware variants). All are supported on the same PCB design through optional component population:

### 5.1 Sensor Set Configuration

| Option | Radar | IMU | ToF | Camera | Use Case |
|--------|-------|-----|-----|--------|----------|
| Minimal | ✅ Acconeer XR112 | ✅ BMI270 | ❌ | ❌ | Stealth deployment, minimal weight |
| Standard | ✅ XR112 or IWR class | ✅ BMI270 | ⚠️ Optional | ❌ | Primary research platform |
| Extended | ✅ Full radar | ✅ BMI270 | ✅ VL53L5CX | ⚠️ Optional | Full capability |

### 5.2 Radio Configuration

| Option | BLE | UWB | Use Case |
|--------|-----|-----|----------|
| BLE-only | ✅ Mandatory | ❌ | Standard operation |
| BLE + UWB | ✅ Mandatory | ✅ DW3000 | Precision timing, moderate bandwidth streaming |

**UWB role options (if populated):**

| Role | Purpose | Bandwidth | Power Impact |
|------|---------|-----------|--------------|
| `timing_only` | Sub-ns synchronization, cm ranging | Minimal | ~50 mW |
| `timing_and_ranging` | Timing + distance measurement | Low burst | ~80 mW |
| `full_bandwidth` | Sustained streaming (e.g., camera) | Up to 5 Mbps | ~100-150 mW |

### 5.3 Camera Configuration (If Populated)

| Option | Module | Resolution | Interface | Power |
|--------|--------|------------|-----------|-------|
| Ultra-low power | HiMax HM01B0 | QVGA | DVP | 2.1 mW |
| Standard | OmniVision OV2640 | 2 MP | DVP | ~100 mW |
| High resolution | Sony IMX219 | 8 MP | MIPI CSI-2 | ~200 mW |

**Camera orientation:** Outward-facing (in the direction the arm is pointing). Not toward the wrist or body.

**Data handling modes (all user-configured):**
- **Metadata-only:** On-device inference, transmit classification only
- **Local storage:** Write frames to SD card per retention policy
- **App streaming:** Encode and stream to companion app via belt node relay
- **Continuous recording:** All data captured and stored

**Optional hardware privacy switch:**
- Physical switch cuts `V_CAM` power rail via MOSFET
- When DISABLED: camera physically unpowered, no software override possible
- PCB provides footprint for switch and MOSFET
- Users who want guaranteed physical disable select this option
- Users who prefer software-only control can leave switch unpopulated

### 5.4 Battery Configuration

| Capacity | Use Case | Runtime Estimate |
|----------|----------|------------------|
| 150 mAh | Minimal config | 48-72 hours |
| 250 mAh | Standard config | 24-48 hours |
| 400 mAh | Extended with camera | 12-24 hours |
| 500 mAh | Extended with UWB + camera | 8-16 hours |

### 5.5 Antenna Diversity (Optional)

**Single BLE radio + Dual antenna = Antenna diversity**

| Option | Antennas | Benefit |
|--------|----------|---------|
| Single antenna | 1 | Standard |
| Antenna diversity | 2 | Improved reliability, body shadowing mitigation |

**Implementation:**
- SPDT RF antenna switch (e.g., Skyworks SKY13330)
- Left-side and right-side antennas on bracelet enclosure
- Radio dynamically selects best antenna based on RSSI

**Benefits:**
- Compensates for arm position changes
- Reduces packet loss from body shadowing
- More deterministic latency for critical alerts
- No self-interference (single transmitter)

**Configuration:**
```toml
[nodes.bracelet_left.antenna_diversity]
enabled = false              # Set true if dual-antenna populated
selection_mode = "rssi"      # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5
```

---

## 6. Sensing Capabilities

### 6.1 mmWave Radar Sensing

**Primary sensing modality for presence and motion detection.**

| Parameter | Acconeer XR112 | TI IWR6843AOP |
|-----------|----------------|---------------|
| Frequency | 60 GHz | 60-64 GHz |
| Range | Up to 15 m | Up to 30 m |
| Resolution | ~5 cm | ~4 cm |
| Field of View | 80° × 40° (typical) | 120° × 120° (AOP) |
| Update rate | Up to 100 Hz | Up to 50 Hz |
| Power (active) | 30-50 mW | 150-300 mW |
| Interface | SPI | SPI |

**Antenna orientation requirement:** Must face outward (palm/dorsal side). Body absorption at 60 GHz is significant; inward-facing antenna would have severely degraded performance.

**Detection capabilities:**
- Human presence at up to 10 m (standard conditions)
- Walking speed estimation via micro-Doppler
- Direction of approach (approaching vs. departing)
- Gesture classification (waving, pointing, etc. — research capability)

### 6.2 IMU Sensing

**Body-frame orientation and arm motion tracking.**

| Parameter | BMI270 | ICM-42688-P |
|-----------|--------|-------------|
| Axes | 6-axis (accel + gyro) | 6-axis |
| Accel range | ±2g to ±16g | ±16g |
| Gyro range | ±125 to ±2000 dps | ±2000 dps |
| Noise density | 0.1 mg | 0.07 mg |
| Interface | SPI/I2C | SPI/I2C |

**Motion classification capabilities:**
- Arm swing detection (gait phase correlation)
- Gesture recognition (research capability)
- Fine motion detection (writing, eating — research capability)

### 6.3 Optional ToF Sensing (Variant B Extended)

**Short-range geometry for close-proximity object detection.**

| Parameter | VL53L5CX |
|-----------|----------|
| Range | Up to 6 m |
| Resolution | 8×8 zone array |
| FoV | 63° diagonal |
| Interface | I2C |
| Use | Close-range geometry, hand proximity |

### 6.4 Optional Camera Sensing (Extended Configuration)

**Arm-direction visual capture.**

**Use cases:**
- Forward-facing visual context
- Object recognition in arm direction
- Evidence capture (user-triggered or detection-triggered)
- Gesture-based interaction (research)

**Data flow:**
```
Camera capture
    │
    ├── On-node inference (metadata-only mode)
    │       └── Classification → BAN → Belt → App
    │
    ├── Local storage (SD card)
    │       └── Raw frames written, `RecordingAvailable` event → BAN → Belt
    │
    └── App streaming (configured)
            └── Encode → BAN → Belt → WiFi → App
```

---

## 7. Haptic Alert System

### 7.1 Directional Haptic Encoding

The haptic actuator on the bracelet provides **directional alerts**. The bracelet that fires indicates the direction of the detected approach relative to the wearer's body.

**Directional mapping:**

| Approach Direction | Primary Haptic Node | Secondary Node |
|-------------------|---------------------|----------------|
| Front-left | Left bracelet | Pendant |
| Left | Left bracelet | Left anklet |
| Rear-left | Left anklet | Left bracelet |
| Rear | Both anklets | — |
| Rear-right | Right anklet | Right bracelet |
| Right | Right bracelet | Right anklet |
| Front-right | Right bracelet | Pendant |

**Implementation:**
- Detection arrives at belt node with body-frame position
- Belt node determines closest node to approach direction
- Belt sends `HapticCommand` BAN message to that node
- Bracelet fires haptic pattern per alert class

### 7.2 Haptic Patterns

| Alert Class | Pattern | Duration | Intensity |
|-------------|---------|----------|-----------|
| HumanApproaching (slow) | Single pulse | 300 ms | Medium |
| HumanApproaching (fast) | Double pulse | 100 ms each, 200 ms gap | Medium-High |
| VehicleNear | Triple pulse with pause | 100 ms each | High |
| GaitAnomaly | Sustained buzz | 500 ms | Medium |
| FallDetected | Extended buzz | 1000 ms, repeat | High |
| ExtremeVelocity | Rapid pulses | 50 ms × 5 | Maximum |

### 7.3 Haptic Hardware

| Component | Specification |
|-----------|---------------|
| Actuator type | LRA (Linear Resonant Actuator) preferred |
| Size | 8-10 mm diameter |
| Resonance | 150-200 Hz typical |
| Driver | TI DRV2605L (I2C, auto-resonance) |
| Waveform library | 123 built-in effects |

**LRA vs ERM:**
- **LRA (recommended):** Faster startup, more precise control, lower power, requires resonant driver
- **ERM (alternative):** Simpler, no resonance tuning, slower response, higher power

---

## 8. Power Architecture

### 8.1 Power Rails

| Rail | Voltage | Source | Usage |
|------|---------|--------|-------|
| `VBAT` | 3.7-4.2V | LiPo battery | Radio PA, haptic peak |
| `VCC_3V3` | 3.3V | LDO or buck | MCU, sensors |
| `VCC_1V8` | 1.8V | LDO | Low-power IMU modes |
| `V_CAM` | 2.8V or 3.3V | Regulated, gated | Camera module (if populated) |
| `V_UWB` | 3.3V | Regulated, gated | UWB module (if populated) |

### 8.2 Power Budget by Configuration

| Configuration | Sleep | Active | Peak |
|---------------|-------|--------|------|
| Minimal (no camera, no UWB) | 5-10 mW | 50-100 mW | 200 mW |
| Standard (no camera) | 10-20 mW | 100-300 mW | 400 mW |
| Extended + Camera | 15-30 mW | 200-500 mW | 700 mW |
| Extended + Camera + UWB | 20-40 mW | 300-700 mW | 900 mW |

**Peak includes:** Haptic actuation at maximum intensity + simultaneous radio transmission.

### 8.3 Power Management States

| State | CPU | Peripherals | BAN Radio | Wake Source |
|-------|-----|-------------|-----------|-------------|
| Active | Full | All on | Connected | N/A |
| Idle | WFI | Sensors active | Connected | Timer, interrupt |
| Sleep | Off (RAM retained) | Minimal | Advertising | Motion detection, timer |
| Deep Sleep | Off | RTC only | Off | External interrupt |

### 8.4 Runtime Estimates

| Configuration | Battery | Estimated Runtime |
|---------------|---------|-------------------|
| Minimal | 150 mAh | 48-72 hours |
| Minimal | 250 mAh | 72-100 hours |
| Standard | 200 mAh | 24-48 hours |
| Standard | 400 mAh | 48-72 hours |
| Extended + Camera | 300 mAh | 12-24 hours |
| Extended + Camera + UWB | 500 mAh | 8-16 hours |

*All values are research targets pending actual measurement.*

---

## 9. BAN Communication

### 9.1 BLE Connection Scheduling

Bracelet nodes participate in the belt-coordinated BLE schedule:

```
BLE Connection Event (30 ms interval, standard mode)
├── Slot 0-2 ms:   Belt → Pendant (sync, commands)
├── Slot 2-5 ms:   Pendant → Belt (sensor data)
├── Slot 5-7 ms:   Belt → Anklet L (sync)
├── Slot 7-10 ms:  Anklet L → Belt (gait event)
├── Slot 10-12 ms: Belt → Anklet R (sync)
├── Slot 12-15 ms: Anklet R → Belt (gait event)
├── Slot 15-17 ms: Belt → Bracelet L (sync)        ← This bracelet
├── Slot 17-20 ms: Bracelet L → Belt (detection)   ← This bracelet
├── Slot 20-22 ms: Belt → Bracelet R (sync)
├── Slot 22-25 ms: Bracelet R → Belt (detection)
├── Slot 25-28 ms: Reserved (critical alerts)
└── Slot 28-30 ms: Reserved (retransmits)
```

**Priority:** Bracelets are priority 4 (lower than anklets and pendant for standard traffic).

### 9.2 QoS Classes

| QoS Class | Priority | Traffic Types | Max Latency |
|-----------|----------|---------------|-------------|
| Critical | Highest | Fall detection, extreme velocity | < 5 ms |
| High | Second | Gait anomaly, stumble precursor | < 20 ms |
| Medium | Third | Presence detection, position tracking | < 50 ms |
| Low | Fourth | Battery status, health telemetry | < 500 ms |

**Bracelet detection events are typically Medium priority.**

### 9.3 UWB Communication (If Populated)

**Roles:**

| Role | Usage |
|------|-------|
| `timing_only` | Precision time sync with belt and other nodes |
| `timing_and_ranging` | Distance measurement to belt for body-frame accuracy |
| `full_bandwidth` | Camera streaming if configured (moderate resolution) |

**UWB is activated only when needed** to conserve power. BLE handles all control and metadata traffic.

---

## 10. RF Coexistence

### 10.1 Frequency Bands

| Radio | Frequency | Conflict Risk |
|-------|-----------|---------------|
| BLE | 2.4 GHz ISM | None (only BLE on bracelet) |
| UWB (optional) | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz | None (separate band) |

**Bracelet RF is simple:** Only BLE (and optional UWB) share no conflict with radar.

### 10.2 Antenna Placement

**BLE antenna:** 
- Integrated chip antenna or PCB trace antenna
- Positioned on outward-facing side of bracelet
- Away from body for best propagation

**mmWave radar antenna:**
- On same outward-facing side
- Minimum 5 mm spacing from BLE antenna
- No metal components between antennas

**Body shadowing consideration:** Wrist position and arm swing significantly affect BLE propagation. Antenna diversity (if populated) mitigates this.

### 10.3 Antenna Diversity (If Populated)

```
Bracelet Enclosure (Top View - Dorsal Side)
┌─────────────────────────────────────────────────────┐
│                                                     │
│   [BLE Antenna #1]              [BLE Antenna #2]   │
│   (Left side)                   (Right side)       │
│                                                     │
│              [mmWave Radar Antenna]                │
│                                                     │
│                     [PCB]                           │
│                                                     │
└─────────────────────────────────────────────────────┘

SPDT Switch selects between Antenna #1 and #2
Radio measures RSSI on both, selects best
```

**Selection algorithm:**
1. Measure RSSI on current antenna
2. Periodically measure RSSI on alternate antenna
3. If alternate is better by > threshold (5 dB), switch
4. Hysteresis prevents rapid switching

---

## 11. Component Reference

### 11.1 MCU Options

| Component | Core | BLE | Flash | RAM | Notes |
|-----------|------|-----|-------|-----|-------|
| nRF5340 | Dual Cortex-M33 | 5.3 | 1 MB | 512 KB | Recommended for firmware consistency |
| STM32WB55 | Cortex-M4 + M0+ | 5.0 | 1 MB | 256 KB | Ultra-low power alternative |
| EFR32BG24 | Cortex-M33 | 5.3 | 1.5 MB | 256 KB | Silicon Labs ecosystem |
| STM32WL55 | Cortex-M4 + M0+ | BLE + LoRa | 256 KB | 64 KB | For extended range research |

### 11.2 mmWave Radar Options

| Component | Freq | Range | Interface | Power | Notes |
|-----------|------|-------|-----------|-------|-------|
| Acconeer XR112 | 60 GHz | 15 m | SPI | 30-50 mW | Ultra-compact, preferred |
| Infineon BGT60ATR24C | 60 GHz | 20 m | SPI | 50-100 mW | Custom antenna design |
| TI IWR6843AOP | 60 GHz | 30 m | SPI | 150-300 mW | Higher resolution, bulkier |

### 11.3 IMU Options

| Component | Noise | Interface | Package | Notes |
|-----------|-------|-----------|---------|-------|
| Bosch BMI270 | 0.1 mg | SPI/I2C | LGA | Preferred for consistency |
| TDK ICM-42688-P | 0.07 mg | SPI/I2C | LGA | Smallest package |
| ST LSM6DSV | 0.08 mg | SPI/I2C | LGA | Built-in gesture/step detection |

### 11.4 Haptic Options

| Component | Type | Size | Resonance | Notes |
|-----------|------|------|-----------|-------|
| Precision Microdrives C10-100 | LRA | 10 mm | 150 Hz | Recommended |
| Precision Microdrives 307-103 | ERM | 8 mm | N/A | Simpler driver |
| AAC ALPS HF2-C3 | LRA | 10 mm | 170 Hz | Ultra-thin (2 mm) |

### 11.5 Optional UWB Module

| Component | Range | Data Rate | Interface | Notes |
|-----------|-------|-----------|-----------|-------|
| Qorvo DW3000 | 200 m | 6.8 Mbps | SPI | Industry standard |
| Qorvo DW300Q | 200 m | 6.8 Mbps | SPI | Compact module |

### 11.6 Optional Camera Modules

| Component | Resolution | Interface | Power | Notes |
|-----------|------------|-----------|-------|-------|
| HiMax HM01B0 | QVGA | DVP | 2.1 mW | Ultra-low power, on-chip detection |
| OmniVision OV2640 | 2 MP | DVP | ~100 mW | Standard resolution |
| Sony IMX219 | 8 MP | MIPI CSI-2 | ~200 mW | High resolution |

---

## 12. Data Handling Configuration

All data handling is user-configured. No system-imposed restrictions.

```toml
[nodes.bracelet_left]
sensors = ["mmwave_radar", "imu", "haptic"]
camera = "none"                  # "none" | "hm01b0" | "ov2640" | "imx219"
uwb = false                      # Enable UWB if module populated
battery_mah = 250
antenna_diversity = false        # Enable if dual-antenna populated

[nodes.bracelet_left.camera]
enabled = false
data_mode = "metadata_only"      # "metadata_only" | "local_storage" | "app_streaming" | "continuous"
store_raw = false
stream_quality = "medium"        # "low" | "medium" | "high" | "raw"

[nodes.bracelet_left.haptic]
enabled = true
default_intensity = 70           # 0-100
patterns = "default"             # "default" | "custom"

[nodes.bracelet_left.antenna_diversity]
enabled = false
selection_mode = "rssi"
switch_threshold_db = 5
```

---

## 13. Firmware Binaries

Two firmware binaries exist for left and right bracelet:

| Binary | Coordinate Mapping | Haptic Direction |
|--------|-------------------|------------------|
| `bracelet_left.rs` | Left arm coordinate frame | Left-side alerts fire this node |
| `bracelet_right.rs` | Right arm coordinate frame | Right-side alerts fire this node |

**Hardware is identical.** Firmware determines handedness.

---

## 14. Mechanical Design Notes

### 14.1 Form Factor

- **Enclosure:** Dorsal-side module (watch case style)
- **Inner surface:** Flat, skin-contact comfortable
- **Sensor apertures:** Outward-facing (radar, optional camera)
- **Haptic actuator:** Internal, vibration transmitted through enclosure
- **Charging contacts:** Inner wrist surface (pogo pins)

**Dimensions (reference):**

| Configuration | Width | Length | Thickness | Weight |
|---------------|-------|--------|-----------|--------|
| Minimal | 35 mm | 25 mm | < 3 mm | 25-40 g |
| Standard | 40 mm | 35 mm | 5-8 mm | 30-50 g |
| Extended | 45 mm | 40 mm | 8-12 mm | 40-60 g |

### 14.2 Band Attachment

- Standard 20-22 mm quick-release spring bars
- Compatible with off-the-shelf watch bands
- Medical-grade silicone or titanium band options

### 14.3 Water Resistance

- IP67 mandatory
- Sweat, rain, hand washing guaranteed
- Sealed enclosure with O-ring gaskets
- Acoustic membrane for microphone (if present)

### 14.4 Materials

**Skin-contact surfaces (EN 1811:2011 compliant):**
- Grade 5 titanium (Ti-6Al-4V)
- 316L surgical stainless steel
- Medical-grade silicone (ISO 10993)
- Hard-coat anodized aluminum

**Non-compliant (do not use):**
- 6061/7075 aluminum alloys
- Unplated brass
- Standard stainless steel

---

## 15. Testing Strategy

### 15.1 Electrical Tests

1. Power-on self-test — MCU boots, firmware CRC verified
2. Rail verification — all rails within ±5% of nominal
3. Sensor enumeration — all I2C/SPI sensors respond
4. Battery verification — voltage 4.1-4.2V on full charge

### 15.2 Functional Tests

5. mmWave radar detection at 0.5 m, 1 m, 3 m
6. IMU gravity vector — reads 9.81 ± 0.2 m/s² at rest
7. Haptic actuation — LRA resonates at specified frequency
8. BAN connectivity — node appears on belt's mesh within 200 ms

### 15.3 Camera Tests (If Populated)

9. Camera initialization
10. Data handling mode verification:
    - Metadata-only: classification tag transmitted, no SD file
    - Local storage: recording file on SD within 500 ms
    - App streaming: belt receives stream, companion app displays
11. Optional hardware switch (if installed): V_CAM = 0V ± 0.05V when DISABLED

### 15.4 UWB Tests (If Populated)

12. UWB module initialization
13. Time sync with belt node — offset < 100 ns
14. Ranging measurement — accuracy ± 10 cm at 1 m

### 15.5 Environmental Tests

15. IP67 immersion — 1 m depth, 30 minutes
16. Post-submersion: powers on, all functions operational
17. Temperature cycling: 0°C to 50°C, verify operation

### 15.6 Wearability Tests

18. Comfort assessment — 8-hour wear study
19. Skin reaction check — hypoallergenic compliance
20. Wrist flexion — no impingement during full ROM

---

## 16. Directory Structure

```
bracelet_node/
├── schematic/
│   ├── bracelet_universal.kicad_pro      # Single PCB design supports all configs
│   └── gerbers/
├── bom/
│   ├── bracelet_minimal.csv               # Minimal population
│   ├── bracelet_standard.csv              # Standard population
│   └── bracelet_extended.csv              # Extended + camera + UWB population
├── test_jig/
│   └── bracelet_test_fixture.kicad_pro
└── README.md
```

---

## 17. Summary Table — Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Standard / Extended | Different component population |
| Camera | None / HM01B0 / OV2640 / IMX219 | Optional module |
| UWB | Disabled / Enabled | DW3000 footprint |
| Antenna diversity | Disabled / Enabled | Dual-antenna footprint |
| Battery | 150 / 250 / 400 / 500 mAh | Different cell sizes |
| Hardware camera switch | Not installed / Installed | Physical power cut |

**All options are configuration choices on the same PCB design.** No separate hardware variants required.
