# Pendant / Necklace Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Neck/chest
**Primary Role:** Upper-hemisphere sensing, acoustic classification, visual identification (opt-in), 360° world capture (curved variant), extreme velocity detection (event-enhanced variant)
**Status:** Reference Design v1.1 — Five Variants
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture

### 1.1 Critical Architectural Constraint

**The pendant node has NO WiFi and NO cellular connectivity.**

This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels. The pendant node cannot connect directly to the companion app, the internet, or any external network.

### 1.2 Data Flow Architecture

All pendant data transmits via Body-Area Network (BAN) to the belt node exclusively. The belt node is the **sole external network gateway** in the SENTINEL-WEAR system.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Pendant Node Connectivity Model                          │
│                                                                              │
│  Pendant Sensors (mmWave, IMU, Camera(s), Acoustic, Environmental)          │
│         │                                                                    │
│         │  On-node processing (compression, event detection)                │
│         ▼                                                                    │
│  BAN Radio (BLE 5.x primary / UWB optional)                                 │
│         │                                                                    │
│         ▼                                                                    │
│  Belt Node (SOLE EXTERNAL GATEWAY)                                          │
│         │                                                                    │
│         ├──► WiFi ──► Companion App (local)                                 │
│         ├──► Cellular ──► Companion App (remote)                            │
│         └──► Storage (SD card, recording management)                        │
│                                                                              │
│  ⚠️  PENDANT NEVER CONNECTS DIRECTLY TO APP OR EXTERNAL NETWORK             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Live Video Streaming Path

When the user has configured live video streaming from a camera-equipped pendant:

```
Camera on Pendant
    │
    ▼
On-pendant encode (H.264/H.265) or raw frame transmission
    │
    ▼
BAN transmission (BLE or UWB)
    │
    ▼
Belt Node receives stream
    │
    ├──► Belt re-encodes if needed
    ├──► Stores to SD card (if recording configured)
    └──► Relays via WiFi/Cellular to Companion App
```

**Key insight:** The pendant creates and transmits video; the belt handles all external routing, storage, and streaming. The pendant has no knowledge of WiFi, cellular, or companion app connections.

### 1.4 BAN Transport Layers

| Transport | Max Bandwidth | When Used | Power |
|-----------|---------------|-----------|-------|
| BLE 5.x | 500 Kbps - 2 Mbps practical | Control, metadata, low-bandwidth data | 5-15 mW |
| UWB | 4-6 Mbps sustained, 6.8 Mbps peak | High-bandwidth continuous, bursts, precision timing | 50-150 mW |

**BLE is present on all pendant variants** (required for baseline operation).

**UWB is optional** on all variants (configuration option at assembly).

### 1.5 Antenna Diversity Consideration

For improved BLE reliability in body-shadowing conditions, the pendant PCB may optionally support dual antennas:

- Primary antenna: Forward-facing
- Secondary antenna: Rear-facing

The MCU selects the antenna with better RSSI dynamically. This is a **configuration option** (antenna population), not a separate hardware variant.

```toml
[ban.antenna_diversity]
enabled = false              # Enable dual-antenna switching
selection_mode = "rssi"      # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5      # Switch if other antenna 5 dB better
```

---

## 2. Overview

The Pendant Node is the highest-information-value wearable node in the SENTINEL-WEAR system. Positioned at chest level, it provides:

- **Primary sensing:** mmWave radar for presence and motion detection
- **Body-frame reference:** IMU for orientation relative to torso
- **Acoustic intelligence:** Microphone array for direction-of-arrival and material classification
- **Visual capture:** Optional cameras (single or 360° array)
- **Environmental context:** Temperature, humidity, VOC, air quality
- **Extreme velocity detection:** Optional event cameras and extended-Doppler radar for fast transient detection

### 2.1 Hardware Variants vs Configuration Options

**Hardware variants** require different PCB designs:
- Different form factors
- Different component arrangements
- Different power architectures
- Different regulatory considerations

**Configuration options** are settings or component population choices on the same PCB:
- Camera module present/absent
- UWB module present/absent
- Battery capacity
- Microphone count
- Component quality tiers

---

## 3. Hardware Variants

### Variant A — Standard Flat Pendant

Compact medallion-style pendant with primary sensing capabilities and optional single camera.

#### Form Factor

| Dimension | Value |
|-----------|-------|
| PCB size | Maximum 50 mm × 50 mm |
| Populated height | < 15 mm |
| Weight | 30–60 g total |
| Water resistance | IP54 minimum; IP67 target |
| Charging | USB-C or magnetic POGO pin |

#### Core Sensing Stack

| Sensor | Purpose | Options |
|--------|---------|---------|
| mmWave Radar | Presence, motion, micro-Doppler | See Section 4.1 |
| IMU | Body-frame orientation | BMI270 or ICM-42688-P |
| Microphone Array | Acoustic DOA, material classification | 3-6 elements |
| Environmental | Atmospheric compensation | BME688 or SEN55 |

#### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Camera | None / Forward-facing / Wide-angle | PCB has footprint, module optionally populated |
| UWB | Disabled / Enabled | PCB has footprint, module optionally populated |
| Battery | 150 / 250 / 300 mAh | Same enclosure, different cell |
| Microphones | 3 / 4 / 6 element | Same footprint, different population |
| Event Camera | None / Forward | For extreme velocity detection |
| Antenna Diversity | Single / Dual | Dual antennas for body-shadowing mitigation |

#### Use Cases

- Basic presence awareness (camera-free configuration)
- Visual capture with streaming (camera configuration)
- Acoustic monitoring and event classification
- Body-frame orientation reference

---

### Variant B — 360° Curved Pendant

Custom curved flexible or rigid-flex PCB following the necklace arc with distributed camera modules. Produces continuous 360° visual coverage from chest level.

**Engineering Note:** The 360° Curved Pendant is architecturally distinct from the Standard Pendant. While listed here as a "variant," it requires completely different PCB design, power architecture, thermal management, and manufacturing processes. It should be treated as a separate node type for engineering purposes.

#### Form Factor

| Dimension | Value |
|-----------|-------|
| Arc length | 120-160 mm |
| Width | 15 mm |
| Thickness | 10-15 mm |
| Weight | 50-120 g (depends on camera count and battery) |
| Water resistance | IP54 minimum |
| Charging | USB-C, magnetic POGO, or belt power input |

#### Camera Coverage Configurations

| Config | Cameras | Angular Spacing | FoV per Camera | Overlap | Bandwidth (VGA H.264) |
|--------|---------|-----------------|----------------|---------|----------------------|
| Minimal | 4 | 90° | ≥ 100° | Adequate | ~1-2 Mbps |
| Standard | 6 | 60° | ≥ 70° | Good | ~1.5-3 Mbps |
| Dense | 8 | 45° | ≥ 55° | Excellent | ~2-4 Mbps |
| Ultra | 12 | 30° | ≥ 40° | Maximum | ~3-6 Mbps |

#### PCB Construction Options

**Option A — Rigid-Flex (Recommended)**
- Rigid PCB segments for components (cameras, processors, connectors)
- Flexible polyimide bridges between segments
- Best reliability and component density
- Minimum bend radius: 3 mm

**Option B — Fully Flexible PCB**
- Single multi-layer flexible substrate
- Better conformability to necklace shape
- More fragile, lower component density
- Suitable for research prototypes

**Option C — Modular Array**
- Individual camera PCBs connected by flex ribbon cables
- Most repairable — individual cameras can be replaced
- More assembly complexity
- Good for iterative development

**Option D — Curved Rigid PCB**
- Standard rigid PCB bent during enclosure assembly
- Only viable for radii > 30 mm
- Limited to larger pendants
- Lower cost, simpler manufacturing

#### Wired Internal Camera Bus Architecture

**The recommended architecture:** All cameras connect via internal MIPI CSI-2 or parallel bus to a central vision processor on the pendant. This provides maximum internal bandwidth without wireless constraints.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    360° Pendant Internal Architecture                        │
│                                                                              │
│   Camera 0 ──┐                                                              │
│   Camera 1 ──┤                                                              │
│   Camera 2 ──┤                                                              │
│   Camera 3 ──┼── MIPI CSI-2 (Internal Bus) ──► Central Vision Processor     │
│   Camera 4 ──┤                                     │                        │
│   Camera 5 ──┤                                     ├── Stitching (on-pendant) │
│   Camera 6 ──┤                                     ├── Encoding (H.264/H.265)│
│   Camera 7 ──┘                                     │                        │
│                                                     ▼                        │
│                                              Single 360° Stream             │
│                                                     │                        │
│                                                     ▼                        │
│                                              BAN Radio (BLE/UWB)            │
│                                                     │                        │
│                                                     ▼                        │
│                                               Belt Node                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Advantages of wired internal bus:**
- No bandwidth constraint within pendant (Gbps internal vs Mbps external)
- Hardware-synchronized capture (FSYNC) easily implemented
- Single encoded output simplifies external transmission
- Lower external bandwidth requirement

#### Hardware Synchronization (FSYNC)

All cameras must capture simultaneously for clean stitching. FSYNC provides hardware-level synchronization:

**Implementation:**
- Single FSYNC GPIO line from vision processor to all camera FSYNC input pins
- PCB trace routing: length-matched to < 10 ns skew between any two cameras
- All cameras trigger on same FSYNC rising edge
- Maximum timing skew: < 1 ms required for acceptable stitching quality

**FSYNC Signal Path:**
```
Vision Processor
    │
    └── FSYNC_GPIO ──┬── Camera 0 FSYNC
                     ├── Camera 1 FSYNC
                     ├── Camera 2 FSYNC
                     ├── Camera 3 FSYNC
                     ├── Camera 4 FSYNC
                     ├── Camera 5 FSYNC
                     ├── Camera 6 FSYNC
                     └── Camera 7 FSYNC
```

**Without FSYNC:** Cameras drift at frame rate relative to each other. Even 1 frame offset (33 ms at 30 fps) causes visible stitching artifacts in dynamic scenes.

#### Camera Modules Per Position

| Sensor Type | Model | Resolution | Interface | Notes |
|-------------|-------|------------|-----------|-------|
| Standard | OV2640 | 2 MP | DVP | Tiny, low cost |
| Standard | OV5640 | 5 MP | MIPI | Autofocus option |
| Low-light | Sony IMX307 | 2 MP | MIPI | Excellent night performance |
| Low-light | Sony IMX327 | 2 MP | MIPI | Similar to IMX307 |
| High-res | Sony IMX219 | 8 MP | MIPI | Raspberry Pi camera sensor |
| High-res | Sony IMX477 | 12 MP | MIPI | Highest quality |
| Ultra-low power | HiMax HM01B0 | QVGA | DVP | 2.1 mW, on-chip detection |
| Event camera | Sony IMX636 | 720p | MIPI | Event + active pixel hybrid |

#### Image Processing Pipeline Options

**Level A — Belt Node Stitching (Recommended for v1)**
- Raw camera streams transmitted via BAN to belt node
- Belt node Linux SoM runs stitching and encoding
- Real-time 360° equirectangular output
- Full-resolution storage on belt SD card
- Companion app accesses 360° view via WiFi from belt
- Lowest pendant complexity and power

**Level B — On-Pendant Stitching**
- Dedicated vision processor (Allwinner V851, NXP i.MX RT1060, or similar) on pendant
- Stitches locally, transmits single compressed 360° stream via BAN
- Higher pendant power, lower belt load
- Better for extended streaming sessions

**Level C — Companion App Stitching**
- Individual compressed camera streams relayed by belt to companion app
- App performs stitching at full quality
- Requires WiFi connection (not suitable for cellular-only or BLE-fallback)
- Highest quality output, highest bandwidth requirement

#### Tiered Progressive Quality Strategy

**Start functional, improve over time. Never blocked waiting for "perfect" data.**

This strategy enables 360° capture even over constrained bandwidth links:

**Phase 1 — Instant Baseline (within 1 second):**
```
All 8 cameras at 320×240 (QVGA) MJPEG
Total bandwidth: ~500 Kbps
Belt receives → stitches rough 360° panorama
User sees: Low-resolution but complete 360° view immediately
Latency: < 1 second from capture to display
```

**Phase 2 — Progressive Refinement (over 5-10 seconds):**
```
Send higher-resolution keyframes sequentially:
  Camera 0: 720p keyframe (~100-200 KB) → belt updates stitching
  Camera 1: 720p keyframe → belt updates stitching
  ...
  Camera 7: 720p keyframe → belt updates stitching

Each keyframe transmitted as burst over 0.5-1 seconds
Combined bandwidth with baseline: ~2-4 Mbps total
Progressive quality improvement visible to user
```

**Phase 3 — Continuous Refinement:**
```
Maintain QVGA baseline stream continuously
Periodically send higher-resolution keyframes
Belt accumulates high-res data over time
Final panorama quality approaches what 720p native would produce
User never sees blank or loading screen
```

**Configuration:**
```toml
[pendant_360.progressive_quality]
enabled = true
baseline_resolution = "qvga"              # "qvga" | "vga" | "720p"
refinement_resolution = "720p"            # Target quality for progressive
refinement_interval_ms = 500              # Time between refinement frames
adaptive_bandwidth = true                 # Adjust based on link quality
min_bandwidth_mbps = 1.0                  # Never drop below this
max_bandwidth_mbps = 5.0                  # Never exceed this
```

#### Bandwidth Capabilities Over UWB

UWB supports continuous streaming at appropriate resolutions:

| Stream Configuration | Bandwidth Required | UWB Capability | Quality Level |
|---------------------|-------------------|----------------|---------------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled | Instant baseline |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled | Standard quality |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled | Good quality |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ✅ At maximum | High quality |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Exceeds capability | Requires WiFi |

**Key insight:** UWB CAN handle all 8 cameras simultaneously — at resolutions appropriate for the bandwidth. The architecture does not prohibit 360° over UWB; it requires selecting appropriate resolution/quality.

#### Vision Processor Options

| Processor | Architecture | Stitching | Encoding | Power | Notes |
|-----------|--------------|-----------|----------|-------|-------|
| Allwinner V851 | ARM Cortex-A7 + ISP | Yes | H.264/H.265 | 0.5-1.5 W | Dedicated ISP |
| NXP i.MX RT1060 | Cortex-M7, 600 MHz | Software | MJPEG | 0.3-0.8 W | No dedicated ISP |
| NXP i.MX 8M Mini | Quad Cortex-A53 + Cortex-M4 | Yes | H.264/H.265 | 1-3 W | Full Linux capable |
| Ambarella CV22 | ARM + NPU | Hardware | H.264/H.265 | 0.5-1 W | Camera SoC |

#### MCU Architecture Options

**Option A — Single High-Performance Processor:**
- NXP i.MX 8M Mini handles: camera multiplexing, stitching, encoding, BAN coordination
- Simplified architecture, single codebase
- Higher power consumption

**Option B — Dual-Processor Split:**
- Sensing MCU: nRF5340 (BAN radio, IMU, radar, acoustic, environmental)
- Vision Processor: i.MX RT1060 or V851 (cameras, stitching, encoding)
- Communication via SPI/UART internally
- Better power management (vision processor can sleep)

**Option C — Research Minimal:**
- nRF5340 for all sensing and BAN
- Raw camera streams transmitted to belt (no on-pendant processing)
- Belt handles all stitching and encoding
- Simplest pendant, highest bandwidth requirement, most belt load

#### Power Architecture

| Component | Active Power | Sleep Power | Notes |
|-----------|--------------|-------------|-------|
| 8 × cameras (active) | 1-2 W | ~0 W (off) | Depends on sensor model |
| Vision processor (stitching) | 1-2 W | 10-50 mW | Duty-cyclable |
| mmWave radar | 100-300 mW | 10-30 mW | Continuous sensing |
| IMU | 10-50 mW | 5-20 mW | Continuous |
| Microphone array | 50-100 mW | 5-20 mW | Event-triggered |
| BLE radio | 10-20 mW | 1-5 mW | Continuous |
| UWB radio | 50-150 mW | 5-20 mW | When active |
| **Total (full active)** | **2.5-4.5 W** | **N/A** | All functions active |
| **Total (reduced duty)** | **1-2 W** | **N/A** | Duty-cycled cameras |
| **Total (sensing only)** | **300-500 mW** | **50-100 mW** | Cameras off |

#### Battery Sizing and Runtime

| Battery | Full Active | Reduced Duty | Sensing Only |
|---------|-------------|--------------|--------------|
| 400 mAh | ~0.5-1 hour | ~1-2 hours | ~4-6 hours |
| 800 mAh | ~1-2 hours | ~2-4 hours | ~8-12 hours |
| 1000 mAh | ~1.5-2.5 hours | ~2.5-5 hours | ~10-15 hours |
| + Belt power input | Unlimited* | Unlimited* | Unlimited* |

*Unlimited with belt power input (conductive chain or cable). Belt battery becomes the constraint.

**Recommendation:** For production use, implement belt power input for extended runtime. Standalone operation suitable for 1-4 hour sessions.

#### Thermal Management

**Problem:** 4W in a 50×50×15mm pendant will exceed comfortable temperature without thermal design.

**Solutions:**

1. **Thermal vias:** Connect vision processor thermal pad to PCB interior layers for heat spreading

2. **Air gap:** Small gap between PCB and enclosure allows air circulation

3. **Thermal interface material:** Thin TIM between vision processor and enclosure for heat transfer

4. **Duty cycling:** Firmware reduces camera frame rate or stitching quality when enclosure temperature exceeds threshold

5. **Larger surface area:** Curved enclosure distributes heat over larger area than flat pendant

**Maximum acceptable temperatures:**
- PCB surface: 70°C (component rating)
- Enclosure exterior (air-facing): 50°C acceptable
- Enclosure interior (skin-contact): 42°C maximum for comfort

**Thermal monitoring:** NTC thermistor placed near vision processor, monitored by MCU. Firmware throttles performance above temperature thresholds.

#### Physical Design

| Element | Specification |
|---------|---------------|
| Enclosure material | Grade 5 titanium channel or 316L surgical steel |
| Front face | Optical-quality transparent polycarbonate at each camera position |
| Inner surface | Medical-grade silicone (skin contact) |
| Weight distribution | Battery distributed along arc or separate chain module |
| Balance | Symmetrical design to prevent rotation |
| Cable exit | Rear or integrated into necklace chain |

---

### Variant C — Medallion (Premium)

Larger pendant with premium materials, higher compute capability, and extended features.

#### Form Factor

| Dimension | Value |
|-----------|-------|
| PCB size | 60 mm × 60 mm or circular 60 mm diameter |
| Thickness | 15-25 mm |
| Weight | 80-150 g |
| Water resistance | IP67 |
| Charging | USB-C PD or wireless Qi |

#### Enhanced Features

| Feature | Specification |
|---------|---------------|
| Compute | NXP i.MX 8M Plus or Raspberry Pi Zero 2W |
| Camera | Single or dual, up to 12 MP |
| Microphone array | 6-element tetrahedral |
| Depth sensing | Optional structured light or time-of-flight |
| Battery | 800-1500 mAh |
| Display | Optional small OLED for status |

#### Use Cases

- Premium personal awareness device
- Executive protection detail
- High-end consumer product
- Research platform requiring more compute

---

### Variant D — Tactical/Extended Runtime

Larger profile pendant designed for all-day operation with belt power input.

#### Form Factor

| Dimension | Value |
|-----------|-------|
| PCB size | 70-80 mm diameter |
| Thickness | 15-25 mm |
| Weight | 100-180 g |
| Water resistance | IP67 |
| Charging | USB-C + belt power input |

#### Key Features

| Feature | Specification |
|---------|---------------|
| Battery | 1500-2000 mAh internal |
| Belt power input | Conductive chain or thin cable |
| Active cooling | Vents or small fan for thermal management |
| Extended operation | 8-12 hours standalone, unlimited with belt power |
| Ruggedized | Impact resistant, wider temperature range |

#### Use Cases

- Security personnel on long shifts
- Journalists in field operations
- Military/law enforcement applications (sensing only, no actuation)
- Emergency response personnel

---

### Variant E — Event-Enhanced

360° pendant variant with dedicated event cameras for extreme velocity detection.

#### Configuration

| Component | Count | Purpose |
|-----------|-------|---------|
| Standard cameras | 6-8 | 360° visual capture |
| Event cameras | 2-4 | Fast transient detection |
| mmWave radar | 1 | Doppler velocity detection |
| IMU | 1 | Body-frame reference |

#### Event Camera Placement

Event cameras are positioned to cover high-probability threat vectors:

**Recommended placement:**
- Forward center (0°): Primary approach detection
- Forward-left (-30° to -45°): Secondary approach
- Forward-right (+30° to +45°): Secondary approach
- Optional rear (180°): Rear approach detection

#### Extreme Velocity Detection Integration

**Doppler Radar Configuration:**
- mmWave radar configured for extended Doppler range (±300+ m/s)
- Standard automotive profiles assume human-scale velocities (±18 m/s)
- Extended configuration trades range resolution for velocity range

**CW Radar for Primary Trigger:**
- Continuous Wave mode provides microsecond-latency detection
- No chirp timing overhead
- Pure Doppler detection: 50-150 µs detection time
- Event camera provides trajectory confirmation

**Event Camera Triggering:**
- Doppler radar detects high-velocity object
- Event camera captures microsecond-resolution trajectory
- Fusion algorithm combines Doppler velocity with event camera angular trajectory
- Detection and alert within < 1 ms

**Latency Budget — Tiered Architecture:**

| Tier | Function | Latency | Sensors |
|------|----------|---------|---------|
| 1 — Reflex Trigger | Binary detection, local alert | 50-150 µs | CW radar |
| 2 — Direction Validation | Velocity vector (magnitude + direction) | 100-500 µs | Event camera |
| 3 — Characterization | Evidence, trajectory prediction | 5-50 ms | FMCW burst, cameras |

**Use Cases:**
- Personal protection in high-threat environments
- Security personnel in active situations
- Research into wearable threat detection

**Note:** This capability detects fast-moving objects. Physical interception of such objects is explicitly out of scope for SENTINEL-WEAR.

#### Realistic Engagement Scenarios

**Rifle at 100 meters:**
- Flight time: 117 ms
- Detection + alert: 0.65-2.4 ms
- Warning time: **115+ ms** — well within human reaction time

**Rifle at 50 meters:**
- Flight time: 58 ms
- Detection + alert: 0.65-2.4 ms
- Warning time: **56+ ms**

**Handgun at 10 meters:**
- Flight time: 28 ms
- Detection + alert: 0.65-2.4 ms
- Warning time: **26+ ms**

**Close-range (< 5 m):**
- Flight time: < 15 ms
- Detection + alert: 0.65-2.4 ms
- Warning time: **Awareness only** — limited time for response

**Key insight:** For realistic engagement distances (rifle at 50+ m), the system provides substantial warning time. Detection range matters more than microsecond-level latency optimization.

---

## 4. Candidate Component Set (All Variants)

### 4.1 mmWave Radar

| Module | Frequency | Range | Interface | Doppler Range | Power | Notes |
|--------|-----------|-------|-----------|---------------|-------|-------|
| TI IWR6843ISK-ODS | 60-64 GHz | 15 m | SPI | Standard: ±18 m/s | 200-400 mW | Evaluation module |
| TI IWR6843 (extended config) | 60-64 GHz | Reduced | SPI | Extended: ±300 m/s | 200-400 mW | Requires custom chirp |
| TI IWR6843 (CW mode) | 60-64 GHz | N/A (range) | SPI | Extended: ±500 m/s | 150-300 mW | CW mode for Tier 1 detection |
| Acconeer XR112 | 60 GHz | 10 m | SPI | Configurable | 30-100 mW | Ultra-compact, preferred for wearable |
| Acconeer XM125 | 60 GHz | 20 m | SPI | Configurable | 50-150 mW | Ultra-long range |
| Infineon BGT60ATR24C | 60 GHz | 10 m | SPI | Configurable | 100-300 mW | MMIC, custom antenna |

**For extreme velocity detection:** TI IWR6843 with extended Doppler configuration or CW mode. Acconeer XR112 for compact wearable applications.

**Antenna orientation:** Forward and lateral hemisphere from chest position. Body absorption significantly attenuates signals facing the torso — outward orientation mandatory.

### 4.2 IMU

| Module | Axes | Noise Density | Interface | Package | Notes |
|--------|------|---------------|-----------|---------|-------|
| Bosch BMI270 | 6-axis | 0.1 mg | SPI/I2C | 2.5×2.5 mm | Preferred |
| TDK ICM-42688-P | 6-axis | 0.07 mg | SPI/I2C | 2.2×2.2 mm | Smallest package |
| ST LSM6DSV | 6-axis | 0.08 mg | SPI/I2C | 2.5×3.0 mm | Built-in step/gesture |
| Bosch BMI323 | 6-axis | 0.12 mg | SPI/I2C | 2.5×2.5 mm | Ultra-low power |

**Critical requirement:** All nodes in SENTINEL-WEAR should use the same IMU model for consistent body-frame fusion. BMI270 is the recommended baseline.

### 4.3 Microphone Array (Acoustic)

**3-element planar (compact, for Standard Pendant):**
- Knowles SPH0645LM4H × 3 (I2S PDM, 65 dBA SNR)
- Inter-microphone spacing: 20-30 mm

**4-element tetrahedral (3D DOA, for Medallion):**
- ST MP23DB01HP × 4 (analog MEMS, 64 dBA SNR)
- InvenSense ICS-40180 × 4 (analog, 65 dBA SNR)

**6-element (high resolution, for Medallion):**
- Knowles SPH0645LM4H × 6
- Circular or tetrahedral arrangement

**Acoustic processing capabilities:**
- Direction-of-arrival (DOA) estimation: ±15° accuracy
- Material classification: glass, ceramic, metal, wood
- Event classification: footstep, door, glass break, voice
- Full-echo profile analysis for material density

### 4.4 Camera Modules

**Single Camera (Standard Pendant, Medallion):**

| Module | Resolution | Interface | Low-light | Autofocus | Power |
|--------|------------|-----------|-----------|-----------|-------|
| OV2640 | 2 MP | DVP/MIPI | Fair | No | 100-150 mW |
| IMX307 | 2 MP | MIPI | Excellent | No | 120-200 mW |
| IMX219 | 8 MP | MIPI | Good | No | 200-400 mW |
| OV5640 | 5 MP | MIPI | Fair | Yes | 200-350 mW |
| IMX477 | 12 MP | MIPI | Good | Yes | 400-700 mW |
| HM01B0 | QVGA | DVP | Fair | No | 2-5 mW (ultra-low power) |

**Event Cameras (for Variant E):**

| Module | Resolution | Interface | Latency | Power | Notes |
|--------|------------|-----------|---------|-------|-------|
| Prophesee EVK3 | 640×480 | USB 2.0 | < 1 ms | 100-300 mW | Evaluation kit |
| Prophesee EVK4 | 1280×720 | USB 3.0 | < 1 ms | 200-500 mW | Higher resolution |
| iniVation DAVIS346 | 346×260 | USB | < 1 ms | 150-300 mW | Frame + events |
| Sony IMX636 | 720p | MIPI | < 1 ms | 50-200 mW | Production module |

**Self-Contained Neural Modules:**

| Module | Capability | Interface | Power |
|--------|------------|-----------|-------|
| Luxonis OAK-D Lite | Stereo + neural inference | USB 3.0 | 2-4 W |
| Luxonis OAK-1 | Neural inference | USB 3.0 | 1-2 W |
| HiMax HM01B0 + on-chip detection | QVGA detection | DVP | 2-5 mW |

### 4.5 Environmental Sensor

| Module | Measurements | Interface | Power |
|--------|--------------|-----------|-------|
| Bosch BME688 | Temp, humidity, pressure, VOC, gas | I2C | 2-5 mW |
| Sensirion SEN55 | PM2.5, PM10, temp, humidity, VOC, NOx | I2C/UART | 50-100 mW |

### 4.6 UWB Module (Optional)

| Module | Range | Data Rate | Interface | Power |
|--------|-------|-----------|-----------|-------|
| Qorvo DW3000 | 10-50 m | 6.8 Mbps max | SPI | 50-150 mW |
| Decawave DWM3000 | 10-50 m | 6.8 Mbps max | SPI | 50-150 mW |

**UWB functions:**
- Sub-nanosecond time synchronization
- Centimeter-level ranging between nodes
- Bandwidth augmentation (moderate streaming, bursts)

**Not for:** Direct companion app connection, WiFi replacement, cellular connectivity.

### 4.7 Power Management

| IC | Function | Notes |
|----|----------|-------|
| Nordic nPM1300 | PMIC, charging, fuel gauge | Recommended for nRF5340-based designs |
| TI BQ25895 | Battery charger | USB-C PD support |
| TI TPS62840 | Buck converter | Ultra-low Iq for always-on |

**Battery specifications:**
- Chemistry: LiPo or Li-Ion
- Voltage: 3.7V nominal (3.0-4.2V range)
- Protection: Integrated over-charge, over-discharge, short-circuit
- Certification: UL 1642 or IEC 62133

### 4.8 Haptic Actuator

| Component | Specification | Notes |
|-----------|---------------|-------|
| Driver IC | TI DRV2605L | I2C, waveform library, back-EMF feedback |
| LRA | Precision Microdrives C10-100 | 10 mm, 150 Hz, 4-5 mm height |
| Alternative | AAC Technologies HF2-C3 | 2 mm height, ultra-thin |

---

## 5. Configuration Options (Same PCB, Different Population)

These options apply to all variants where the PCB design supports the option. Different population does NOT create a new hardware variant.

### 5.1 Camera Configuration

| Option | PCB Requirement | Notes |
|--------|-----------------|-------|
| None | Camera footprint unpopulated | Sensing-only configuration |
| Forward-facing | Camera footprint populated, standard module | Single forward view |
| Wide-angle | Camera footprint populated, wide-angle module | Increased coverage |
| Event camera | Event camera footprint populated | For extreme velocity detection |

**Configuration at assembly or by user:** PCB designed with camera footprint; module can be populated at manufacture or added later by user with appropriate skills.

### 5.2 UWB Configuration

| Option | PCB Requirement | Notes |
|--------|-----------------|-------|
| Disabled | UWB footprint unpopulated | BLE-only operation |
| Enabled | UWB module populated | Timing, ranging, bandwidth augmentation |

**Impact on power budget:** UWB adds 50-150 mW when active. Firmware manages duty cycling to conserve power.

### 5.3 Battery Capacity

| Option | Enclosure Requirement | Notes |
|--------|----------------------|-------|
| Standard | Standard enclosure | 150-300 mAh |
| Extended | Same or slightly larger | 400-600 mAh |
| Maximum | Larger enclosure variant | 800-1500 mAh |

**Configuration at assembly:** Same PCB, different battery cell installed.

### 5.4 Microphone Count

| Option | PCB Requirement | Notes |
|--------|-----------------|-------|
| 3-element | 3 microphone footprints populated | Compact, basic DOA |
| 4-element | 4 microphone footprints populated | Tetrahedral 3D DOA |
| 6-element | 6 microphone footprints populated | High-resolution |

**Configuration at assembly:** PCB supports all options; population determines capability.

### 5.5 Antenna Diversity

| Option | PCB Requirement | Notes |
|--------|-----------------|-------|
| Single antenna | Primary antenna populated | Standard |
| Dual antenna | Both antennas populated | Body-shadowing mitigation |

---

## 6. Interface Summary

### 6.1 Common Interfaces (All Variants)

| Interface | Signal Names | Direction | Connection |
|-----------|--------------|-----------|------------|
| mmWave Radar | `SPI_MOSI`, `SPI_MISO`, `SPI_CLK`, `RADAR_CS`, `RADAR_IRQ` | SPI + GPIO | Radar module |
| IMU | `IMU_CS`, `IMU_INT` | SPI + GPIO | IMU module |
| Microphone Array | `PDM_CLK`, `PDM_DATA` or `I2S_BCLK`, `I2S_WS`, `I2S_DATA` | I2S/PDM | Microphones |
| Environmental | `ENV_SDA`, `ENV_SCL` | I2C | Environmental sensor |
| BAN Radio | Integrated in MCU | BLE 5.x | Antenna |
| Charging | `CHRG_IN+`, `CHRG_IN-` | Power | USB-C/POGO |
| Debug | `SWD_CLK`, `SWD_DIO`, `UART_TX`, `UART_RX` | Debug | 4-pin header |

### 6.2 Standard/Medallion Additional Interfaces

| Interface | Signal Names | Direction | Connection |
|-----------|--------------|-----------|------------|
| Camera | `MIPI_CLK+`, `MIPI_CLK-`, `MIPI_D0+`, `MIPI_D0-` | MIPI CSI-2 | Camera module |
| Privacy Switch (optional) | `CAM_PWR_SW` | GPIO | Hardware switch |
| Haptic | `HAPTIC_SDA`, `HAPTIC_SCL` | I2C | DRV2605L |
| UWB (optional) | `UWB_MOSI`, `UWB_MISO`, `UWB_CLK`, `UWB_CS`, `UWB_IRQ` | SPI | DW3000 |

### 6.3 360° Pendant Additional Interfaces

| Interface | Signal Names | Direction | Connection |
|-----------|--------------|-----------|------------|
| Camera × N | `CAM_n_CLK+`, `CAM_n_CLK-`, `CAM_n_D0+`, `CAM_n_D0-` | MIPI CSI-2 per camera | Each camera |
| Camera FSYNC | `CAM_FSYNC` | GPIO output | All camera FSYNC pins |
| Vision MCU Link | `USB3_TX+`, `USB3_TX-`, `USB3_RX+`, `USB3_RX-` | USB 3.0 or HSIC | Vision processor |
| Belt Power (optional) | `V_BELT_IN`, `V_BELT_GND` | Power input | Conductive chain/cable |

**MIPI routing requirements:**
- Differential impedance: 100 Ω ± 10%
- Length match within pair: < 2 mil (0.05 mm)
- Intra-pair spacing: Minimum per MIPI spec
- Inter-pair spacing: 3× trace width minimum
- Via count: Minimize, matched per pair

---

## 7. Power Architecture

### 7.1 Standard Pendant Power Domains

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Standard Pendant Power Architecture                       │
│                                                                              │
│   V_BAT (3.7-4.2V) ─── LiPo Battery                                          │
│        │                                                                     │
│        ├──► V_PA (unregulated) ──► BLE PA                                   │
│        │                                                                     │
│        └──► nPM1300 PMIC                                                     │
│                 │                                                            │
│                 ├──► V_SYS (3.3V) ──► MCU, sensors, BLE radio              │
│                 │                                                            │
│                 └──► V_CAM (3.3V/2.8V, switched) ──► Camera (if populated) │
│                              │                                               │
│                              └──► Optional MOSFET for privacy switch       │
│                                                                              │
│   Charging: USB-C or POGO ──► nPM1300 ──► Battery                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Rail | Voltage | Source | Usage | Current (typical) |
|------|---------|--------|-------|-------------------|
| V_BAT | 3.7-4.2V | LiPo | Raw battery | System total |
| V_PA | 3.7-4.2V | V_BAT | BLE/UWB PA | 10-50 mA |
| V_SYS | 3.3V | nPM1300 | MCU, sensors | 50-150 mA |
| V_CAM | 3.3V/2.8V | nPM1300, switched | Camera | 100-400 mA |

### 7.2 360° Pendant Power Domains

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      360° Pendant Power Architecture                         │
│                                                                              │
│   V_BAT (3.7-4.2V) ─── LiPo (400-1000 mAh)                                  │
│        │                                                                     │
│        ├──► V_PA (unregulated) ──► BLE/UWB PA                               │
│        │                                                                     │
│        ├──► Buck Regulator                                                   │
│        │        │                                                            │
│        │        └──► V_SYS (3.3V) ──► MCU, radar, IMU, BLE                 │
│        │                                                                     │
│        ├──► LDO Regulator                                                    │
│        │        │                                                            │
│        │        └──► V_CAMS (3.3V/2.8V) ──► All camera modules             │
│        │                 │                                                   │
│        │                 └──► FSYNC driver                                  │
│        │                                                                     │
│        └──► Vision Processor Supply                                          │
│                 │                                                            │
│                 └──► V_ISP (1.8V, 1.0V) ──► Vision processor               │
│                                                                              │
│   V_EXT (5V, optional) ─── Belt Power Input                                  │
│        │                                                                     │
│        └──► Boost Regulator ──► V_BAT (supplement/replace battery)          │
│                                                                              │
│   Charging: USB-C or POGO ──► Charger IC ──► Battery                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Rail | Voltage | Source | Usage | Current (typical) |
|------|---------|--------|-------|-------------------|
| V_BAT | 3.7-4.2V | LiPo | Main battery | System total |
| V_SYS | 3.3V | Buck | MCU, radar, IMU, BLE | 100-300 mA |
| V_CAMS | 3.3V/2.8V | LDO, ganged | All N cameras | 200-800 mA |
| V_ISP | 1.8V, 1.0V | Vision processor supply | Vision processor | 200-1000 mA |
| V_EXT | 5V | Belt power input (optional) | Extended runtime | 500-1500 mA |

### 7.3 Power Consumption Summary

| Mode | Standard Pendant | 360° Pendant (8-camera) | Notes |
|------|------------------|-------------------------|-------|
| Sleep | 5-15 mW | 10-30 mW | Radio advertising, sensors off |
| Sensing only | 150-300 mW | 300-500 mW | Radar, IMU, acoustic active |
| Sensing + camera | 300-700 mW | — | Standard pendant with camera |
| Full active | — | 2500-4500 mW | All cameras + stitching + streaming |
| Reduced duty | — | 1000-2000 mW | Duty-cycled cameras |

### 7.4 Runtime Estimates

| Battery | Standard Pendant (sensing) | 360° Pendant (full active) | 360° Pendant (reduced duty) |
|---------|---------------------------|---------------------------|----------------------------|
| 150 mAh | 6-10 hours | — | — |
| 300 mAh | 12-20 hours | — | — |
| 400 mAh | — | 0.5-1 hour | 1-2 hours |
| 800 mAh | — | 1-2 hours | 2-4 hours |
| 1000 mAh | — | 1.5-2.5 hours | 2.5-5 hours |

**With belt power input:** Unlimited runtime (belt battery becomes constraint).

---

## 8. Thermal Management

### 8.1 Standard Pendant

Power dissipation: 300-700 mW maximum

**Thermal design:**
- Standard PCB with copper pours for heat spreading
- No special thermal management required
- Enclosure temperature typically < 35°C at skin contact

### 8.2 360° Pendant

Power dissipation: 2.5-4.5 W maximum

**Critical thermal design requirements:**

1. **Thermal vias under vision processor:** 0.3 mm diameter, 1 mm pitch, connect top to internal ground plane

2. **Heat spreader:** Copper layer on PCB interior for distribution

3. **Air gap:** 1-2 mm gap between PCB and enclosure inner surface

4. **Thermal interface material:** Optional thin TIM between vision processor and enclosure

5. **Active ventilation (Variant D):** Small vents in enclosure for air circulation

**Temperature monitoring:**
- NTC thermistor (10K β=3380) placed near vision processor
- Connected to MCU ADC
- Firmware monitors temperature, throttles performance above thresholds

**Thermal throttling behavior:**
```
Temperature < 40°C: Full performance
Temperature 40-45°C: Reduce camera frame rate, stitch quality
Temperature 45-50°C: Disable non-essential cameras, reduce encoding quality
Temperature > 50°C: Vision processor sleep, sensing-only mode
```

---

## 9. Manufacturing Notes

### 9.1 Standard Pendant Manufacturing

- Standard rigid PCB process
- 4-layer minimum (signal + ground + power + signal)
- 1.6 mm thickness standard
- ENIG finish for wire bonding compatibility
- SMT assembly for all components

### 9.2 360° Pendant Manufacturing (Rigid-Flex)

**Layer stack example (8-camera):**
- 8 rigid sections: Each 15-20 mm × 15-20 mm
- 7 flex bridges: Each 5 mm × 20-40 mm
- Total length: 120-160 mm

**Design rules:**
- Minimum bend radius: 3 mm (polyimide flex)
- Flex thickness: 0.1-0.2 mm
- Rigid thickness: 0.8-1.6 mm
- Coverlay on flex sections for protection

**MIPI routing on rigid sections:**
- Differential impedance: 100 Ω ± 10%
- Length match within pair: < 2 mil
- No vias on differential pairs (or matched via pairs)
- Ground guard traces between camera pairs

**Camera module attachment:**
- COB (chip-on-board) for lowest profile
- Module connector for repairability
- FPC (flexible printed circuit) tail for each camera

### 9.3 Quality Control Points

| Check | Standard Pendant | 360° Pendant |
|-------|------------------|--------------|
| Electrical continuity | 100% | 100% |
| Camera initialization | If populated | All N cameras |
| FSYNC timing | N/A | ±1 ms skew check |
| Power consumption | Within 10% spec | Within 15% spec |
| Thermal validation | N/A | Max temp < 50°C under load |
| Water ingress (IP67) | Yes, if rated | Yes, if rated |

---

## 10. Testing Strategy

### 10.1 All Variants — Mandatory Tests

| Test | Procedure | Pass Criteria |
|------|-----------|---------------|
| Power-on self-test | Apply power, verify boot | MCU boots, LED indicates status |
| IMU initialization | Read WHO_AM_I register | Correct device ID returned |
| IMU gravity vector | Read accelerometer at rest | 9.81 ± 0.2 m/s² |
| Radar detection | Place target at 0.5 m | Detection within 500 ms |
| BAN connectivity | Ping belt node | Latency < 10 ms, packet loss < 1% |
| Battery level | Read fuel gauge | 4.1-4.2V on full charge |
| Charging verification | Apply charger, measure current | Current within spec |
| Firmware version | Query MCU | Matches expected build hash |

### 10.2 Camera Tests (If Populated)

| Test | Procedure | Pass Criteria |
|------|-----------|---------------|
| Camera initialization | Init sequence | MIPI link established within 2 s |
| Frame capture | Request frame | Valid frame within 200 ms |
| Data handling verification | Test each mode | Metadata/local storage/streaming per config |

### 10.3 360° Pendant Tests

| Test | Procedure | Pass Criteria |
|------|-----------|---------------|
| All cameras initialize | Init all N cameras | All N respond within 5 s |
| FSYNC timing | Measure trigger skew | All cameras within ±1 ms |
| Stitching quality (belt) | Stitch test chart | Straight lines continuous, seam < 3 px |
| 360° recording | Record 10 seconds | File on belt SD, valid format |
| Companion app view | Request 360° stream | Display within 5 s via belt WiFi |
| Thermal test | Run 30 min full active | Enclosure temp < 50°C |

### 10.4 Optional Hardware Switch Test (If Installed)

| Test | Procedure | Pass Criteria |
|------|-----------|---------------|
| DISABLED state | Set switch to disabled, measure V_CAM | V_CAM < 0.1 V |
| ENABLED state | Set switch to enabled, measure V_CAM | V_CAM > 3.0 V |
| Firmware state | Query MCU for switch GPIO | Matches physical switch position |

---

## 11. Directory Structure

```
pendant_node/
├── variant_a_standard/
│   ├── pendant_std.kicad_pro
│   ├── pendant_std.kicad_sch
│   ├── pendant_std.kicad_pcb
│   ├── bom/
│   │   └── pendant_std_bom.csv
│   └── gerbers/
│       └── (production files)
│
├── variant_b_360_curved/
│   ├── pendant_360_main.kicad_pro
│   ├── pendant_360_segment.kicad_sch
│   ├── pendant_360_flex_bridge.kicad_sch
│   ├── pendant_360_full_assembly.kicad_pcb
│   ├── pendant_360_stitching_config.json
│   ├── bom/
│   │   └── pendant_360_bom.csv
│   ├── 3d_models/
│   │   └── (STEP/STL for enclosure)
│   └── gerbers/
│       ├── rigid_sections/
│       └── flex_bridges/
│
├── variant_c_medallion/
│   ├── medallion.kicad_pro
│   ├── medallion.kicad_sch
│   ├── medallion.kicad_pcb
│   ├── bom/
│   └── gerbers/
│
├── variant_d_tactical/
│   ├── tactical.kicad_pro
│   ├── tactical.kicad_sch
│   ├── tactical.kicad_pcb
│   ├── bom/
│   └── gerbers/
│
├── variant_e_event_enhanced/
│   ├── event_enhanced.kicad_pro
│   ├── event_enhanced.kicad_sch
│   ├── event_enhanced.kicad_pcb
│   ├── bom/
│   └── gerbers/
│
├── shared/
│   ├── sensor_footprints.pretty/
│   └── mechanical/
│
├── bom/
│   └── components_master.csv
│
├── 3d_models/
│   ├── pendant_std.step
│   ├── pendant_360_arc.step
│   └── medallion.step
│
└── README.md
```

---

## 12. Related Documentation

| Document | Location | Purpose |
|----------|----------|---------|
| 360° Pendant Architecture | `docs/theory/360_pendant_architecture.md` | Technical deep-dive |
| Extreme Velocity Sensing | `docs/theory/extreme_velocity_sensing.md` | Doppler + event camera fusion |
| Doppler Radar Configuration | `docs/theory/doppler_radar_config.md` | Extended velocity detection |
| BAN Bandwidth Budget | `docs/theory/ban_bandwidth_budget.md` | BLE/UWB allocation strategies |
| Power Management | `docs/theory/power_management.md` | Battery and power strategies |
| Thermal Management | `docs/theory/thermal_management.md` | Wearable thermal design |
| Hardware Configuration | `hardware/hardware_config.md` | Interface standards, pin maps |
| Firmware Architecture | `firmware/firmware.md` | Embedded firmware design |
| Calibration Guide | `docs/guides/calibration_360_pendant.md` | 360° calibration procedure |
| Deployment Paths | `docs/guides/deployment_paths.md` | Configuration guidance |

---

## 13. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024-01-XX | Initial release |
| 1.1 | 2024-XX-XX | Added Variant D (Tactical) and Variant E (Event-Enhanced); corrected UWB bandwidth capabilities; added wired internal bus architecture; added tiered progressive quality strategy; expanded thermal management section; added realistic engagement distance analysis |
