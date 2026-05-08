# SENTINEL-WEAR — Distributed Body-Frame Wearable Sensing Mesh

**An open research framework for wearable, jewelry-form-factor distributed sensors that build a 360° body-coordinate awareness field around the wearer. Full environmental capture. Full world reconstruction. Full user control.**

[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](#license)
[![License: CERN OHL](https://img.shields.io/badge/Hardware-CERN%20OHL--S%20v2-green.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Sensing-Only Effectors](https://img.shields.io/badge/Effectors-Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

SENTINEL-WEAR is a **research and education** project studying the architecture and feasibility of wearable distributed sensing in jewelry form factors — necklaces, bracelets, anklets, belt-pieces, eyewear — that together build a body-frame awareness field around the wearer. The project is motivated by the observation that personal protective adornments (literal or symbolic) are present across nearly every culture in human history, and that with modern miniaturized sensors, the *sensing* dimension of personal protection is now genuinely feasible in those form factors.

**Strictly limited to:**
- Sensor architecture (mmWave radar, IMU, low-power LiDAR, acoustic, environmental, event cameras, conventional cameras, 360° imaging).
- Distributed perception in body-frame coordinates.
- Tracking and threat-classification algorithms (PentaTrack).
- Alerting, haptic, and information-display research.
- Form-factor and human-factors research (comfort, weight, battery, charging, durability, water resistance, hypoallergenic materials).
- Full world reconstruction — both sparse probabilistic models and dense SLAM-based 3D maps.
- Opt-in visual identification for any node class via camera integration.
- Feasibility research on detection physics at body-frame timescales.
- Storage, streaming, and legal evidence capture of all sensor data.

**Explicitly out of scope:**
- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any mechanism designed to deflect, intercept, capture, or otherwise act physically against incoming objects.
- Any active protection system actuator design at any scale.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

---

## What the Project Actually Builds

A **distributed wearable sensor mesh** worn on the body that continuously maintains a body-frame 3D awareness field. The wearer becomes the origin of their own coordinate system; sensors at multiple body locations cover overlapping hemispheres around the body. The system supports two parallel world-model modes simultaneously:

**Mode A — Sparse Probabilistic World Model:** PentaTrack-based, low-power, event-driven. Fast, efficient. Answers: "Something is here, moving this way, at this speed."

**Mode B — Dense SLAM World Map:** Full 3D reconstruction from LiDAR + camera arrays. Geometry-complete. Answers: "This is what the environment looks like, with objects identified and tracked." Requires higher compute (belt node Linux SoM variant or companion app offload).

Both modes run simultaneously when hardware supports it. Mode A always runs. Mode B activates based on hardware configuration and user preference.

```
                         ┌─────────────────────────────────┐
                         │     Eyewear Node (Optional)      │
                         │  Event Cam + IMU + optional cam  │
                         │  forward hemisphere, head-stable  │
                         └──────────────┬──────────────────┘
                                        │
             ┌──────────────────────────┼────────────────────────────┐
             │                          │                            │
   ┌─────────▼────────────┐  ┌──────────▼──────────────┐  ┌─────────▼──────────┐
   │  Pendant / Necklace  │  │  Pendant / Necklace      │  │  Pendant Standard  │
   │  Node — Standard     │  │  Node — 360° Curved      │  │  Node — Medallion  │
   │  Flat pendant form   │  │  Curved flexible PCB     │  │  Premium housing   │
   │  mmWave+IMU+Acoustic │  │  8-camera 360°+mmWave    │  │  mmWave+IMU+Acous  │
   │  + opt-in cam        │  │  +IMU+Acoustic+Radar     │  │  +opt-in full cam  │
   └─────────────────────┘  └─────────────────────────┘  └────────────────────┘
                                        │
                          ┌─────────────┴──────────────┐
                          │                            │
             ┌────────────▼────────┐     ┌────────────▼────────┐
             │  Bracelet Left      │     │  Bracelet Right     │
             │  mmWave+IMU+Haptic  │     │  mmWave+IMU+Haptic  │
             │  + opt-in cam       │     │  + opt-in cam       │
             └─────────────────────┘     └─────────────────────┘
                                        │
                             ┌──────────▼──────────┐
                             │      Belt Node       │
                             │  PRIMARY HUB         │
                             │  SOLE EXTERNAL GATEWAY│
                             │  Compute+Battery     │
                             │  IMU(Torso Ref)      │
                             │  WiFi+Cellular+BAN   │
                             │  App Server + API    │
                             │  Recording Manager   │
                             │  SLAM Processor      │
                             └──────────┬───────────┘
                                        │
                          ┌─────────────┴──────────────┐
                          │                            │
             ┌────────────▼────────┐     ┌────────────▼────────┐
             │  Anklet Left        │     │  Anklet Right       │
             │  ToF/LiDAR+IMU+     │     │  ToF/LiDAR+IMU+     │
             │  Haptic+opt cam     │     │  Haptic+opt cam     │
             └─────────────────────┘     └─────────────────────┘
```

---

## Connectivity Architecture — The Belt Node as Sole External Gateway

### Critical Architectural Principle

**Only the Belt Node connects to external networks.** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via Body-Area Network (BAN) to the belt node.** They have no WiFi, no cellular, and no direct connection to the companion app.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SENTINEL-WEAR Connectivity Model                         │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  ⚠️  BELT NODE IS THE ONLY NODE WITH WiFi / Cellular / BLE (to external)    │
│  ⚠️  ALL OTHER NODES = BAN ONLY (BLE/UWB to belt, nothing external)          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Wearable | BLE on Wearable | UWB on Wearable |
|------------|------------------|-----------------|-----------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Form factor impact** | Requires larger battery, antenna space | Minimal | Moderate |
| **Wearability** | Breaks jewelry form factor | Preserves jewelry form | Compatible |
| **Determinism** | Variable latency, contention | Consistent, schedulable | Excellent timing |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power/thermal/size requirements. BLE and UWB are the only viable BAN technologies for jewelry-scale wearables.

### Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE 5.x (500 Kbps - 2 Mbps practical)                                       │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, IMU orientation, classification results)            │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on, lowest power (5-15 mW)                                         │
│  → Present on ALL nodes                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  UWB (4-6 Mbps sustained, 6.8 Mbps peak)                                     │
│  → Precision time synchronization (sub-nanosecond)                          │
│  → Precision ranging (cm-level distance between nodes)                       │
│  → Moderate-bandwidth continuous (single camera at 720p-1080p)               │
│  → High-bandwidth BURSTS (compressed clips, short recordings)                │
│  → 360° at 2K-2.5K resolution                                                │
│  → All 8 cameras at QVGA-VGA simultaneously                                  │
│  → Optional on nodes, enabled per configuration                              │
│  → Power: 50-150 mW when active                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — BELT NODE ONLY                                        │
│  → High-bandwidth continuous (4K 360° live streaming)                        │
│  → Multiple simultaneous camera streams                                      │
│  → SLAM data transfer                                                        │
│  → Primary path to companion app                                             │
│  → Power: 300-500 mW when streaming                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — BELT NODE ONLY                              │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips                │
│  → LTE Cat 4: 50-150 Mbps — good for moderate streaming                     │
│  → 5G: 100-1000+ Mbps — suitable for 360° live relay                         │
│  → Used when WiFi unavailable (remote access, away from home)               │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### UWB Bandwidth Capabilities — Detailed

UWB supports continuous streaming at appropriate resolutions:

| Stream Type | Bandwidth Required | UWB Capability | Notes |
|-------------|-------------------|----------------|-------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled | Instant baseline for 360° |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled | Standard quality for all cameras |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled | Good quality panorama |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ✅ At max | High quality panorama |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Exceeds | Requires WiFi |

**Tiered Progressive Quality Strategy:** When bandwidth is constrained, the system starts with low-resolution baseline and progressively improves:

```
Phase 1 (Instant):     8 cameras at QVGA → complete 360° view immediately
Phase 2 (Progressive): Send higher-res keyframes sequentially over 5-10 seconds
Phase 3 (Continuous):  Maintain low-res baseline, periodic high-res keyframes
Result:                Functional immediately, quality improves over time
```

### Supported Cellular Modules (Belt Node Only)

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |

### Configuration (`sentinel-wear.toml`)

```toml
[connectivity]
# WiFi is primary connection to companion app
wifi_enabled = true
wifi_ssid = "YourHomeNetwork"
wifi_password = ""  # Set at runtime for security

# Cellular is optional for remote access
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Stream video over cellular (data cost)
emergency_contact = ""           # Phone/endpoint for emergency alerts

# BAN configuration
[connectivity.ban]
primary = "ble"                  # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = false              # Enable UWB on nodes that support it
uwb_role = "timing_and_ranging"  # "timing_only" | "timing_and_ranging" | "full_bandwidth"
```

---

## Body-Area Network (BAN) Architecture

### BLE Scheduling and Determinism

**Problem:** If all nodes transmit simultaneously over BLE, collisions and latency spikes occur.

**Solution — Time-slotted scheduling:**

```
BLE Connection Event (30 ms interval)
├── Slot 0-2 ms:   Belt → Pendant (sync, commands)
├── Slot 2-5 ms:   Pendant → Belt (sensor data, detections)
├── Slot 5-7 ms:   Belt → Anklet L (sync)
├── Slot 7-10 ms:  Anklet L → Belt (gait event)
├── Slot 10-12 ms: Belt → Anklet R (sync)
├── Slot 12-15 ms: Anklet R → Belt (gait event)
├── Slot 15-17 ms: Belt → Bracelet L (sync)
├── Slot 17-20 ms: Bracelet L → Belt (detection)
├── Slot 20-22 ms: Belt → Bracelet R (sync)
├── Slot 22-25 ms: Bracelet R → Belt (detection)
├── Slot 25-28 ms: RESERVED (critical alerts, extreme velocity)
└── Slot 28-30 ms: Reserved (retransmits, eyewear)
```

**Connection interval tradeoffs:**

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation |
| 100 ms | ~60 ms | 2-5 mW | Power saving |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### QoS Classes and Priority

**Traffic is prioritized by class:**

| QoS Class | Priority | Traffic Types | Latency Target |
|-----------|----------|---------------|----------------|
| Critical | Highest (preempts all other traffic) | Extreme velocity detection, fall detection, emergency alerts | < 5 ms |
| High | Second (preferential queue) | Gait anomaly, stumble precursor | < 20 ms |
| Medium | Third (normal scheduling) | Presence detection, position tracking | < 50 ms |
| Low | Fourth (bandwidth available) | Battery status, health telemetry | < 500 ms |

**Critical QoS behavior:**
- Messages tagged Critical bypass normal scheduling
- Transmit immediately, preempting ongoing transmissions
- Reserved slot (25-28 ms) guaranteed every cycle for critical alerts
- Essential for extreme velocity detection latency requirements

### Dynamic Scheduling

**Context-aware scheduling adjusts based on activity:**

| Context | Schedule Adjustment |
|---------|---------------------|
| Walking detected | Prioritize anklets (gait timing critical) |
| Threat detected | Prioritize pendant (sensing critical) |
| Idle state | Extend intervals (power saving) |
| Streaming active | Shift heavy traffic to UWB |
| Extreme velocity alert | Preempt all other traffic |

**Configuration:**

```toml
[ban.scheduling]
mode = "dynamic"               # "static" | "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true
extreme_velocity_preempt = true
```

### Node Priority for Bandwidth Allocation

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

### Antenna Diversity (Belt Node and Optionally Other Nodes)

**Problem:** Human body absorbs and blocks 2.4 GHz RF significantly. Torso absorption: 10-30 dB. Arm position changes: 5-15 dB variation.

**Solution:** Single BLE radio with multiple antenna paths. Radio dynamically selects best antenna.

```
Belt Node BLE Radio
    │
    ├── Antenna Switch
    │       ├── Left-side antenna (toward wearer's left)
    │       └── Right-side antenna (toward wearer's right)
    │
    └── Radio selects antenna with best RSSI
```

**Benefits:**
- Reduced packet loss from body shadowing
- Lower retries → deterministic latency
- Reliable transmission regardless of body orientation
- Critical for extreme velocity detection reliability
- No self-interference (only one transmitter active at a time)

**Why antenna diversity, not multiple BLE radios:**

| Approach | Self-Interference | Complexity | Power | Benefit |
|----------|-------------------|------------|-------|---------|
| Multi-radio BLE | High (desense, coupling) | High | High | Marginal bandwidth gain |
| Antenna diversity | None | Low | Low | Reliability improvement |
| Dynamic scheduling | None | Firmware only | Zero | Latency improvement |
| QoS classes | None | Firmware only | Zero | Critical traffic priority |

**Multi-radio BLE is NOT recommended for SENTINEL-WEAR:**
- RF complexity exceeds benefits at this scale (4-6 nodes)
- Self-interference from closely-spaced radios
- Antenna coupling and desense problems
- Increased power consumption
- Increased firmware complexity

**Antenna diversity IS recommended:**
- Single radio, multiple antenna paths
- No self-interference
- Improved reliability without RF chaos
- Appropriate for wearable scale

**Configuration:**

```toml
[ban.antenna_diversity]
enabled = true
antenna_count = 2              # 1 = single, 2 = diversity
selection_mode = "rssi"        # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5        # Switch if other antenna is 5 dB better
preferred_antenna = "auto"     # "left" | "right" | "auto"
```

### UWB Role Configuration

UWB can be enabled on any node with UWB hardware. Roles:

**Timing Only:**
- Sub-nanosecond time synchronization
- cm-level ranging between nodes
- Minimal data transfer
- Lowest power UWB mode

**Timing + Burst Data:**
- All timing capabilities
- Short bursts of compressed data
- Useful for: compressed camera clips, sensor data bursts
- Power: activated only during burst

**Full Bandwidth:**
- All timing capabilities
- Sustained streaming up to 5-6 Mbps
- Useful for: single camera streaming, 360° at 2K resolution
- Power: higher sustained draw (50-150 mW)

**Configuration:**

```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"     # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false      # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5          # Conservative limit for thermal management
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]
```

### RF Coexistence

**Frequency bands in use:**
- BLE: 2.4 GHz ISM band
- WiFi: 2.4 GHz and 5 GHz bands
- UWB: 3.1-10.6 GHz
- mmWave radar: 60 GHz
- Cellular: Various (700 MHz - 3.5 GHz, 24-47 GHz for 5G mmWave)

**Coexistence strategies:**
1. **WiFi prefers 5 GHz band** — eliminates 2.4 GHz conflict with BLE
2. **Antenna separation on belt** — Cellular/WiFi antennas on exterior, BLE/UWB on interior
3. **Time-division multiplexing** — When 2.4 GHz WiFi must be used, coordinate timing with BLE
4. **UWB is in separate band** — No conflict with BLE or WiFi

```toml
[rf_coexistence]
wifi_prefer_5ghz = true        # Strongly recommended
ble_wifi_timeshare = false     # Set true if 2.4 GHz WiFi must be used
ble_gap_during_wifi_ms = 5     # Gap between BLE and WiFi if sharing
```

---

## Node Architecture — All Variants

### Pendant / Necklace Node

The highest-information-value node. Hardware variants and configuration options:

#### Hardware Variants (Require Different PCB)

**Variant A — Standard Flat Pendant**
- Flat PCB in a medallion-style enclosure
- mmWave radar (forward + lateral hemisphere)
- IMU (body-frame orientation)
- Microphone array (acoustic DOA + material classification)
- 150–300 mAh battery
- ~30–60 g total

**Variant B — 360° Curved Pendant** (Treat as separate node type for engineering purposes)
- Custom curved flexible PCB or rigid-flex following the necklace arc
- 4–8 camera modules distributed around the circumference
- mmWave radar arrays at multiple angular positions
- Distributed IMU and microphone arrays
- 400–800 mAh battery (distributed in pendant arc or separate module)
- Wired internal camera bus to central vision processor
- ~50–120 g depending on camera count and battery

**Variant C — Medallion (Premium)**
- Larger pendant with thicker profile
- Houses higher-compute local processor
- Higher-resolution cameras (single or dual)
- More microphones (6-element tetrahedral array)
- Integrated LiDAR / structured light
- High-capacity battery (up to 1000 mAh)
- ~70–120 g total

**Variant D — Tactical/Extended Runtime**
- Larger profile (70-80 mm)
- Integrated battery pack in enclosure (up to 2000 mAh)
- External belt power input (conductive chain or cable)
- Purpose: All-day 360° operation for security personnel, journalists
- Thermal: Active cooling vents, larger surface area

**Variant E — Event-Enhanced**
- 360° camera array (standard or reduced count)
- Plus dedicated event cameras for extreme velocity detection
- Purpose: Users who need the full sensing stack including fast transient detection
- Configuration: Standard cameras for visual + event cameras for Doppler-triggered capture

**Variant F — Long-Range Detection**
- Higher-power CW radar (dedicated module, not AOP)
- Optimized antenna for range
- Trade range resolution for maximum detection distance
- Standard event camera for direction
- Purpose: Maximize detection range for rifle detection at 100+ meters
- Target: Detect rifle fire at 150+ meters with 150+ ms warning time

#### Configuration Options (Same PCB, Different Population/Settings)

For Standard and Medallion variants, these are configuration options (not separate hardware variants):

| Option | Values | Notes |
|--------|--------|-------|
| Camera | None / Forward-facing / Wide-angle | PCB has footprint, module optionally populated |
| UWB | Disabled / Enabled | PCB has footprint, module optionally populated |
| Battery | 150 / 250 / 400 mAh | Same enclosure, different cell |
| Microphones | 3 / 4 / 6 element | Same footprint, different population |
| Extended Doppler | Disabled / Enabled | For extreme velocity detection |

For 360° Curved variant, these are configuration options:

| Option | Values | Notes |
|--------|--------|-------|
| Camera count | 4 / 6 / 8 | Same PCB design, different population |
| Camera resolution | QVGA / VGA / 720p / 1080p | Same cameras, different mode |
| Stitching location | Pendant / Belt | Vision processor location |
| UWB | Disabled / Enabled | For bandwidth augmentation |

### Bracelet Node (×2)

Forearm-hemisphere sensing and directional haptic output.

#### Hardware Design

**Single PCB design** supports all variants via configuration:

- nRF5340 or STM32WB55 MCU
- Acconeer XR112 mmWave radar (outward-facing)
- BMI270 or ICM-42688-P IMU
- TI DRV2605L haptic driver + LRA actuator
- 150-500 mAh battery (configurable)
- Optional UWB footprint (DW3000)
- Optional camera footprint

#### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Standard / Extended | Different component population |
| Camera | None / Forward-facing | Optional module |
| UWB | Disabled / Enabled | Optional module |
| Battery | 150 / 250 / 400 mAh | Different cell sizes |

### Belt Node

Primary compute, battery hub, torso reference, **sole external network gateway**, app server.

#### Hardware Variants (Different Compute Platforms)

**Variant A — MCU-Class (Minimal)**
- STM32H7 class (Cortex-M7, 480 MHz)
- Local embedded RTOS
- Runs `sentinel-belt-controller` Rust binary
- Sparse tracking only (no SLAM)
- ~2000 mAh battery
- 12-20 hour runtime (sparse mode)
- Suitable for: basic presence awareness, low-cost deployment

**Variant B — Linux SoM (Standard)**
- Raspberry Pi CM4 / NXP i.MX 8M Plus
- Full Linux stack
- SLAM processing, companion app server, recording management
- 5000-7000 mAh battery bank
- 10-15 hour runtime (sparse mode), 5-8 hours (full active)
- Suitable for: full capability deployment, 360° processing

**Variant C — High-Performance**
- Qualcomm SA8155P or NXP i.MX 8M Plus with NPU
- GPU-class inference for dense SLAM
- Neural 360° stitching from pendant cameras locally
- Enables full offline dense world reconstruction
- 5000-7000 mAh battery bank
- 5-8 hours runtime (full active)
- Suitable for: production deployment, maximum capability

**Variant D — ESP32-Based**
- ESP32-S3 with integrated WiFi
- Lowest cost option
- WiFi used only as BAN transport to edge-like processing
- Sufficient for 2-3 nodes
- ~2000 mAh battery
- Suitable for: minimal deployment, research

**Variant E — Extended Runtime/Hot-Swappable**
- Dual battery bays with hot-swap capability
- User can swap one battery while running on the other
- 24/7 operation for security, shift workers
- Larger form factor (belt pack style)
- Suitable for: professional/continuous use

#### Configuration Options (All Variants)

| Option | Values | Notes |
|--------|--------|-------|
| Cellular module | None / LTE Cat 1 / LTE Cat 4 / 5G | Based on remote access needs |
| UWB | Disabled / Enabled | For precision timing with nodes |
| WiFi band preference | 5 GHz preferred / 2.4 GHz only | For RF coexistence |
| Battery capacity | 2000 / 5000 / 7000 mAh | Based on variant and enclosure |
| Antenna diversity | Disabled / Enabled | Dual BLE antennas |

#### Haptic Configuration (All Variants)

| Option | Values | Notes |
|--------|--------|-------|
| Haptic type | LRA / Piezo / Both | Piezo provides faster onset (0.5-1 ms) |
| Directional haptics | Disabled / Enabled | Multiple actuators for threat direction |
| Actuator count | 1 / 2 / 4 | Around belt circumference |

**Piezo vs LRA haptics:**
- LRA onset: 2-10 ms
- Piezo onset: 0.5-1 ms
- For extreme velocity detection at close range (< 15 m), piezo provides faster alert
- For rifle detection at 100+ m, LRA is sufficient (flight time is 117+ ms)

#### Power Budget Analysis

| Mode | Functions Active | Power Draw | Runtime (5000 mAh) |
|------|------------------|------------|-------------------|
| Minimal (MCU-only) | BAN hub, sparse tracking | ~500 mW | ~37 hours |
| Standard (Linux) | BAN hub, sparse tracking, WiFi idle | ~1.4 W | ~13 hours |
| Full Active | BAN hub, dense SLAM, 360° stitching, streaming | ~4.5 W | ~4 hours |
| Remote Streaming | Full Active + cellular | ~5.3 W | ~3.5 hours |

#### Thermal Management

Linux SoM variants generate 3-7 W under full load. Requirements:
- Thermal pad under SoM footprint
- Vented or heat-spreader enclosure
- NTC thermistor for thermal monitoring
- Firmware throttles SLAM frame rate above 40°C enclosure temperature
- Maximum skin-contact temperature: 42°C

### Anklet Node (×2)

Ground-plane sensing, gait analysis, lower-hemisphere coverage.

#### Hardware Design

**Single PCB design** supports all variants via configuration:

- Nordic nRF5340 or Silicon Labs EFR32BG24 MCU
- VL53L5CX ToF or short-range LiDAR
- BMI270 IMU (high dynamic range for heel-strike)
- TI DRV2605L haptic driver + LRA
- 300-500 mAh battery
- Optional UWB footprint
- Optional small camera footprint

#### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal (ToF only) / Extended (ToF + LiDAR + mmWave) | Different population |
| Camera | None / Downward-facing | Ground obstacle detection |
| UWB | Disabled / Enabled | Recommended for gait sync |
| Battery | 300 / 500 mAh | Different cell sizes |

#### Gait Analysis Capabilities

- Step frequency (cadence)
- Step regularity (gait coefficient of variation)
- Heel-strike impact magnitude (requires high-dynamic-range IMU, 5-10 g)
- Stride asymmetry (left vs. right comparison)
- Pre-stumble signature detection
- Gait phase: stance / swing / strike / push at ≥ 200 Hz

### Eyewear Node (Optional)

Head-stabilized forward-hemisphere sensing.

#### Hardware Variants

**Clip-On Design (Variants A, B):**
- Single PCB, clips to existing glasses
- 30 mm × 20 mm × 8 mm
- < 8 g total
- 50-80 mAh battery
- Suitable for: research, intermittent use

**Frame-Integrated Design (Variant C):**
- PCB segments embedded in temple arms
- 60 mm × 4 mm × 3 mm per arm
- Requires custom or semi-custom frames
- 80-150 mAh battery (in temples or separate module)
- Suitable for: permanent deployment

**Headband Design (Alternative to Frame-Integrated):**
- Central PCB at forehead
- Camera modules on flex extensions at temporal positions
- Easier to prototype than frame-integrated
- Same capability as frame-integrated

#### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor type | Event-only / Event + Camera | Event-only for fast transient detection |
| Camera count | 1 / 2 / 3 | Forward + side cameras |
| UWB | Disabled / Enabled | For precision head-torso sync |
| Power source | Battery / Belt cable (Variant C) | Cable for extended runtime |

---

## World Model Architecture

SENTINEL-WEAR supports two simultaneous world model modes. Both are always available; hardware configuration determines the fidelity of each.

### Mode A — Sparse Probabilistic Body-Centric World Model

**Technology stack:** OMNI-SENSE → PentaTrack → Body-frame stabilizer

**What it produces:**
```
TrackedEntity {
  id
  position (x, y, z)     // in body frame
  velocity (vx, vy, vz)  // in body frame
  prediction_centers     // PentaTrack center field
  confidence
  classification_hint    // "human", "vehicle", "animal"
  anomaly_flags
}
```

**Visualization:** Radar-style body-centric display. Detected entities shown as blobs with velocity vectors and PentaTrack prediction arcs. Confidence shown as opacity/size. Alert zones shown as colored rings.

**Use cases:** All alert generation, gait analysis, directional haptic routing, low-latency real-time awareness.

**Power:** Runs on all hardware configurations. Always active. ~100-300 mW.

### Mode B — Dense SLAM World Map

**Technology stack:** LiDAR + cameras → SLAM → 3D mesh → Object recognition → Tracked entities overlay

**What it produces:**
```
WorldMap {
  dense_3d_mesh         // full geometry of environment
  object_instances      // detected and classified objects
  tracked_entities      // same as Mode A but geometry-anchored
  camera_feeds          // per-camera streams with entity overlays
  stitched_panorama     // 360° view from pendant cameras (if 360° variant)
}
```

**Visualization:** Full 3D world reconstruction. Users can fly-through the captured environment, review recordings in 3D context, overlay tracked entities on the geometry.

**Use cases:** Full environment review, legal evidence, SLAM-based odometry for body-frame enhancement, 360° panoramic recording.

**Power:** Requires belt node Linux SoM variant or companion app compute offload. ~1-3 W.

### Dual Mode Simultaneous Operation

When belt node is Linux SoM class, both modes run simultaneously:
- Mode A is the real-time alert and tracking backbone (always active, low latency)
- Mode B builds in background for rich review, replay, and export
- Mode A benefits from Mode B: object classifications from SLAM improve PentaTrack drift profiles
- Mode B benefits from Mode A: PentaTrack predictions annotate the dense map with motion intent

### Mobile SLAM Characteristics

Unlike room-mounted SLAM (stationary sensors), SENTINEL-WEAR SLAM is **mobile** — sensors move with the wearer:

**Challenges:**
- Motion blur during walking (pendant sway)
- Revisitation detection (how to recognize "same kitchen")
- Map segmentation (home vs office vs transit)
- Memory constraints for persistent maps

**Solutions:**
- IMU-based image stabilization (software compensation)
- Visual loop closure when revisiting locations
- User-controlled map segments (can delete specific locations)
- Background optimization for large maps

---

## 360° Curved Pendant — Technical Architecture

The 360° curved pendant is a signature node type of SENTINEL-WEAR. It transforms the pendant from a directional sensor to a continuous 360° capture platform.

### Wired Internal Camera Bus Architecture

**Recommended approach:** All cameras connect via internal MIPI CSI-2 or parallel bus to a central vision processor on the pendant. This provides:
- Maximum internal bandwidth (no wireless constraint within pendant)
- Hardware-synchronized capture (FSYNC)
- Single encoded stream to belt node

```
8 Cameras → MIPI CSI-2 (internal) → Vision Processor on Pendant
                                           │
                                           ├── Stitch on-pendant → Single 360° stream
                                           └── Encode (H.264/H.265)
                                                   │
                                                   ▼
                                              UWB or WiFi* to Belt
                                                   │
                                                   ▼
                                           Belt → WiFi → Companion App

*WiFi only if pendant has WiFi module (optional, not recommended for jewelry form factor)
```

### Camera Coverage Configurations

| Config | Cameras | Angular Spacing | FoV Needed | Bandwidth (VGA H.264) |
|--------|---------|-----------------|------------|----------------------|
| Minimal | 4 | 90° | ≥ 100° | ~1-2 Mbps |
| Standard | 6 | 60° | ≥ 70° | ~1.5-3 Mbps |
| Dense | 8 | 45° | ≥ 55° | ~2-4 Mbps |
| Ultra | 12 | 30° | ≥ 40° | ~3-6 Mbps |

### Bandwidth Capabilities Over UWB

UWB supports continuous streaming at appropriate resolutions:

| Stream Type | Bandwidth | UWB Capability | Notes |
|-------------|-----------|----------------|-------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled | Instant baseline |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled | Standard quality |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled | Good quality |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ⚠️ At max | High quality |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Exceeds | Requires WiFi |

### Tiered Progressive Quality Strategy

**Start functional, improve over time. Never blocked waiting for "perfect" data.**

```
Phase 1 (Instant — within 1 second):
    Send all 8 cameras at 320×240 (QVGA) MJPEG
    Total: ~500 Kbps
    Belt stitches rough 360° panorama immediately
    User sees: Low-res but complete 360° view

Phase 2 (Progressive — over 5-10 seconds):
    Send higher-res keyframes sequentially:
    Camera 0: 720p keyframe → belt updates stitching
    Camera 1: 720p keyframe → belt updates stitching
    ...
    Camera 7: 720p keyframe → belt updates stitching
    Each keyframe: ~100-200 KB, sent over seconds
    Combined with baseline: ~2-4 Mbps total

Phase 3 (Continuous):
    Maintain QVGA baseline continuously
    Periodically send higher-res keyframes
    Final panorama quality approaches what 720p would produce
```

### Hardware Synchronization (FSYNC)

All cameras must capture simultaneously for clean stitching:
- Single FSYNC GPIO line from vision processor to all camera FSYNC pins
- All cameras trigger on same FSYNC rising edge
- PCB trace length matching: target < 10 ns skew between any two cameras
- Without FSYNC: cameras drift at frame rate, causing visible stitching artifacts

### Power Architecture

| Component | Power Draw | Notes |
|-----------|------------|-------|
| 8 × cameras active | 1-2 W | Depends on sensor model |
| Vision processor (stitching) | 1-2 W | ARM-based ISP or dedicated chip |
| Radar + IMU + acoustic | 0.5 W | Continuous sensing |
| BLE/UWB radio | 0.1-0.2 W | Transmission to belt |
| **Total active** | **2.5-4.5 W** | Full 360° streaming |
| **Total (reduced duty)** | **1-2 W** | Standard quality with duty cycling |

**Battery implications:**
- 800 mAh at 3.7V = ~3 Wh
- At 3W draw: ~1 hour runtime
- At 1.5W draw (duty-cycled): ~2 hours runtime
- Recommendation: Use belt power input for extended operation

### Thermal Management

4W in a 50×50×15mm pendant will exceed comfortable temperature:
- Thermal vias to spread heat away from vision processor
- Air gap between PCB and enclosure for dissipation
- Maximum skin-contact temperature: 40-42°C
- Duty cycling to reduce sustained thermal load

---

## Extreme Velocity Sensing — Production and Research

### Production Capability

The extreme velocity sensing architecture (Doppler radar + event camera fusion) is designed for **production use**, not just research. This enables detection of fast-moving objects (bullets, debris, fragments) within the body-frame awareness field.

### Realistic Engagement Distances

**Key insight:** The original analysis over-constrained to 3-meter "point-blank" scenarios. Realistic engagements occur at much greater distances with significantly more warning time.

**Detection and warning at realistic distances:**

| Weapon | Velocity | Distance | Flight Time | Detection+Alert | Warning Time |
|--------|----------|----------|-------------|-----------------|--------------|
| Rifle | 850 m/s | 50 m | 58.8 ms | 0.65-2.4 ms | **56-58 ms** |
| Rifle | 850 m/s | 100 m | 117.6 ms | 0.65-2.4 ms | **115-117 ms** |
| Rifle | 850 m/s | 200 m | 235.3 ms | 0.65-2.4 ms | **233-235 ms** |
| Handgun | 350 m/s | 10 m | 28.6 ms | 0.65-2.4 ms | **26-28 ms** |
| Handgun | 350 m/s | 15 m | 42.9 ms | 0.65-2.4 ms | **40-42 ms** |

**Implications:**
- At 100+ meters (rifle), warning time is 115+ ms — well within human reaction time
- At 50+ meters (rifle), warning time is 56+ ms — sufficient for reflex response
- At 10-15 meters (handgun), warning time is 26-42 ms — awareness but limited response time
- LRA haptics (2-10 ms onset) are sufficient for realistic rifle distances
- Piezo haptics (0.5-1 ms onset) provide additional margin for close-range scenarios

### Detection Range vs Latency

**The primary optimization target is detection range, not detection latency.**

- Detection latency (50-150 µs) is trivial compared to any realistic flight time
- Detection range determines warning time margin
- Longer detection range = more warning time
- Latency optimization is valuable but secondary to range optimization

### Sensor Requirements

**Doppler Radar (mmWave):**
- Standard automotive profiles assume human-scale velocities (±18 m/s)
- Extended configuration required for projectile detection (±300+ m/s)
- TI IWR6843 can be configured for extended Doppler range with reduced range resolution
- CW (Continuous Wave) mode preferred for zero scanning latency

**Event Camera:**
- Microsecond-latency motion detection (10-100 µs, not 1 ms)
- Triggers on rapid angular motion across pixel array
- Provides trajectory vector for approaching object

### Tiered Detection Architecture

**Tier 1 — Reflex Trigger (50-150 µs):**
- CW radar threshold detection
- Local haptic trigger
- Purpose: Immediate awareness

**Tier 2 — Direction Validation (100-500 µs):**
- Event camera streak analysis
- Direction estimation
- Purpose: Directional alert routing

**Tier 3 — Characterization (5-50 ms):**
- FMCW burst for range + velocity
- Evidence capture
- Purpose: Forensic analysis, logging

### Hardware Configuration

```toml
[extreme_velocity]
enabled = false                 # Opt-in feature
mode = "production"             # "disabled" | "research" | "production"

[extreme_velocity.detection]
# Optimization target
optimization_target = "range"   # "range" | "latency" | "balanced"

# Range-optimized configuration
target_range_m = 150           # Detect at 150 m for 175+ ms warning

# CW radar parameters
radar_mode = "continuous_wave"
fft_window_us = 32             # 32 µs window sufficient for projectile Doppler
velocity_threshold_ms = 50     # Ignore below 50 m/s

# QoS for alerts
qos_class = "critical"         # Preempts all other BLE traffic
reserved_slot = true
```

### Integration with 360° Pendant

The 360° pendant can integrate event cameras:
- **Variant E — Event-Enhanced:** 6 conventional + 2 event cameras
- Event cameras positioned at key angles (e.g., forward ± 30°)
- Conventional cameras provide color/detail; event cameras provide velocity detection
- **Variant F — Long-Range Detection:** Higher-power radar for 150+ m detection

---

## Sensor Stack

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        SENTINEL-WEAR Sensing Stack                           │
├──────────────────────────────────────────────────────────────────────────────┤
│  Always-On Layer (Low Power — All Nodes)                                     │
│  mmWave radar (presence, motion, micro-Doppler activity signatures)          │
│  IMU (body-frame orientation, gait, gesture)                                 │
│  PIR (optional — binary presence at specific nodes)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│  High-Fidelity Layer (Event-Triggered or Continuous)                         │
│  Solid-state LiDAR / ToF (geometry, ground clearance, obstacle detection)    │
│  Event-based camera (microsecond-latency fast object detection)              │
│  Microphone array (acoustic DOA, material classification, event detection)   │
│  Environmental sensors (temperature, humidity, VOC, air quality)             │
├──────────────────────────────────────────────────────────────────────────────┤
│  Visual Capture Layer (User-Configured — Any Node)                           │
│  Conventional cameras at any angular position on any node                    │
│  360° curved pendant multi-camera array (signature variant)                  │
│  Full video recording (raw, compressed, continuous, or on-trigger)           │
│  Live streaming to companion app                                              │
│  SLAM-ready visual odometry output                                            │
├──────────────────────────────────────────────────────────────────────────────┤
│  Extreme Velocity Layer (Production-Capable)                                 │
│  Doppler radar (CW mode, extended velocity range)                            │
│  Event camera (microsecond detection)                                        │
│  Fusion for projectile detection within realistic engagement windows          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Software Architecture

### Crate Structure

```
sentinel-wear/
├── crates/
│   ├── sentinel-core/              # Shared types, config, node descriptors, events
│   ├── sentinel-perception/        # Per-node sensor pipelines (OMNI-SENSE based)
│   ├── sentinel-body-frame/        # Multi-IMU body-frame fusion, stabilization, calibration
│   ├── sentinel-fusion/            # Multi-node detection fusion (CI + JPDA)
│   ├── sentinel-tracking/          # PentaTrack bridge, body-space prediction
│   ├── sentinel-extreme-velocity/  # High-speed / sprint / collision prediction
│   ├── sentinel-slam/              # SLAM pipeline, 360° stitching, dense world model
│   ├── sentinel-ban-protocol/      # BAN protocol (BLE 5.x / UWB)
│   ├── sentinel-alerts/            # Alert classification, haptic encoding, routing
│   ├── sentinel-storage/           # Recording management, retention, export
│   ├── sentinel-api/               # Companion app REST/WebSocket/media API server
│   └── sentinel-belt-controller/   # Main binary: runs on belt node
├── firmware/                       # no_std embedded firmware per node type
├── hardware/                       # PCB schematics, BOMs, test jig
│   ├── schematic/
│   │   ├── pendant_node/           # Standard and medallion variants
│   │   ├── 360_pendant_node/       # 360° curved pendant (separate node type)
│   │   ├── bracelet_node/
│   │   ├── belt_node/
│   │   ├── anklet_node/
│   │   └── eyewear_node/
│   ├── testing/
│   │   └── test_jig_pcb/
│   └── hardware_config.md
├── apps/                           # Companion app architecture and API spec
├── docs/                           # Theory, guides, API reference
│   ├── guides/
│   │   ├── getting_started.md
│   │   ├── body_frame_calibration.md
```
│   │   ├── alert_modalities.md
│   │   ├── calibration_360_pendant.md
│   │   ├── power_profiles.md
│   │   └── connectivity_configuration.md
│   ├── theory/
│   │   ├── future_research.md
│   │   ├── body_coordinate_fusion.md
│   │   ├── drift_profiles_at_body_scale.md
│   │   ├── form_factor_human_factors.md
│   │   ├── extreme_velocity_sensing.md
│   │   ├── doppler_radar_config.md
│   │   ├── slam_world_model.md
│   │   ├── 360_pendant_architecture.md
│   │   ├── ban_bandwidth_budget.md
│   │   ├── power_management.md
│   │   ├── thermal_management.md
│   │   ├── rf_coexistence.md
│   │   ├── passive_materials_research.md
│   │   └── why_no_actuation.md
│   ├── api/
│   │   └── api_spec.md
│   └── protocol/
│       └── protocol_spec.md
├── scenarios/                      # Simulation scenarios
├── mechanical/                     # CAD, STL, enclosure designs
├── production/                     # Manufacturing SOPs, QC
├── legal/                          # Compliance, ethics, export control
├── config/                         # Configuration file templates
│   └── sentinel-wear-full.toml
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

### Crate vs Firmware Relationship

**Crates** run on the belt node (compute hub):
- Fusion algorithms (multi-node data combining)
- PentaTrack (predictive tracking)
- SLAM (dense world model)
- API server (companion app interface)
- Recording management
- Alert routing

**Firmware** runs on each node's MCU:
- Sensor drivers (OMNI-SENSE)
- On-node processing (compression, event detection)
- BAN protocol (BLE/UWB communication)
- Power management
- Local storage (if configured)

**Data flow:** Sensors → Firmware (node MCU) → BAN → Crate (belt node) → WiFi/Cellular → Companion App

---

## Time Synchronization

Distributed wearable sensing requires precise clock alignment across all body nodes.

**Belt node as time master:** Broadcasts synchronized timestamps to all nodes at each BLE connection event interval.

**Per-node offset estimation:** Each node runs `omni-sense-time::ClockOffsetEstimator` (PTP-style four-timestamp exchange) tracking local clock drift with continuous correction.

**UWB upgrade path:** Qorvo DW3000-class UWB available on all nodes for sub-nanosecond ranging and time alignment when required for SLAM accuracy or extreme velocity detection.

**360° pendant sync:** The 360° curved pendant requires tight synchronization across all camera modules for clean stitching:
- Camera sync lines within the pendant PCB provide hardware-level sync
- The belt node time master provides cross-pendant timing reference
- Target: < 10 ns skew between any two cameras

---

## Body-Frame Coordinate System

The belt node's IMU defines the body-frame origin:
- **+X:** Wearer's right
- **+Y:** Wearer's forward
- **+Z:** Up (gravity direction inverted)

All node detections are expressed in this stabilized frame via `omni-sense-frames::BodyFrameStabilizer`. Stabilization mode (World, Translational, None) configurable per application.

**Translational stabilization (default):** Body frame co-translates with wearer but does not co-rotate. "Approach from right rear" means actual approach from that direction, regardless of which way the wearer is facing.

Walk-through calibration (`sentinel-body-frame::WalkThroughCalibrator`) establishes geometric offsets of each node relative to the torso origin. The 360° pendant calibration includes angular offset of each camera module relative to the pendant center.

**Head-torso separation:** When eyewear node is present, head orientation is computed separately from torso orientation, enabling correct attribution of forward-facing detections regardless of head position.

---

## Privacy Model — User Configured

SENTINEL-WEAR has no system-imposed data restrictions. All data handling is user-configured:

**Privacy controls available (all user-selected):**
- Physical power switch on any camera module (user cuts power when desired)
- Software disable mode (camera driver disabled in firmware config)
- Activity-based recording (record only on detection trigger)
- Schedule-based recording (e.g., off during sleep)
- No recording (metadata-only mode)

**No mandatory kill switch requirement.** The architecture supports hardware switches for users who want them. It also supports software-only privacy controls for users who prefer them. Both are valid configurations. The system does not mandate either approach.

**Data handling options (all user-configured in `sentinel-wear.toml`):**
- Metadata-only transmission (classification results only)
- Raw local storage (SD card, internal flash)
- App streaming (live or on-demand)
- Full continuous recording
- Tiered: metadata by default, raw on alert
- Any combination of the above

```toml
[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "sd_card"       # "sd_card" | "internal_flash" | "network_path"
retention_days = 30              # 0 = keep forever
continuous_recording = false
recording_trigger = "on_detection"  # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0               # 0 = unlimited
include_integrity_chain = true
```

---

## Data Storage and Companion App

### Storage Options

All node classes support local storage:
- microSD card (recommended — removable for physical access, up to 2 TB)
- Internal flash (non-removable, smaller capacity)
- Network storage via belt node Wi-Fi

### Companion App

Mobile (iOS/Android) and desktop (Windows/macOS/Linux) apps connect to the belt node's embedded server.

**Connection modes:**
- **Primary:** Wi-Fi (same network as belt node)
- **Remote:** Cellular (when away from home network)
- **Fallback:** BLE direct (limited to metadata and alerts)

**Core features:**
- Live sensor display (both sparse world model and dense SLAM view)
- 360° panoramic live view from curved pendant cameras
- Full recordings library with playback, search, timeline scrubbing
- 3D replay with time-scrubbing through the SLAM world model
- Gait analytics, event history, anomaly logs
- System configuration
- Legal export with cryptographic integrity chain

**Remote access:** When enabled, companion app connects via belt node's cellular or remote WiFi endpoint. Requires user-set bearer token. All communication encrypted.

See `apps/README.md` for full API and architecture specification.

---

## Companion App World Model Views

### View 1 — Radar / Event Display (always available)

Body-centric radar-style display:
- Wearer at center
- Detected entities as blobs with velocity vectors
- PentaTrack prediction arcs
- Alert zones as colored rings
- Entity classification labels
- Confidence as opacity
- Gait event annotations

### View 2 — 360° Live Panorama (360° pendant variant)

Equirectangular or spherical 360° live view from pendant cameras:
- All camera streams stitched in real-time
- Entity detections overlaid as bounding boxes
- Works in playback for recordings review
- Supports VR headset viewing

### View 3 — Dense SLAM 3D World Map (Linux SoM belt variant)

Interactive 3D world model:
- Full geometry of every visited environment
- Objects classified and labeled
- Tracked entity trajectories overlaid
- Time-scrubbing to replay any moment in 3D context
- Useful for forensic review, legal preparation, security analysis

---

## Output Modalities

What SENTINEL-WEAR does with what it perceives:

- **Haptic alerts:** Directional buzz on the appropriate node — "approach from right rear" buzzes the right-rear node.
- **Audio alerts:** Bone conduction or earpiece (optional).
- **Companion-app overlay:** Live world model on phone, tablet, or desktop.
- **360° live view:** From curved pendant cameras to any connected display.
- **Recordings:** All data stored locally per user configuration. Available for review, export, legal use.
- **Legal evidence export:** Cryptographic hash chain, timestamp, device ID embedded in exported files.
- **Emergency contact:** Manually triggered (never automatic) location + clip share to designated contact.

### Alert Routing

| Alert Class | Haptic Pattern | Priority | Nodes Involved |
|-------------|---------------|----------|----------------|
| HumanApproaching slow | Single pulse, 300ms | Info | Nearest to approach direction |
| HumanApproaching fast | Double pulse, 100ms each | Warning | Nearest to approach direction |
| VehicleNear | Triple pulse with pause | Warning | Pendant + nearest bracelet |
| GaitAnomaly | Sustained buzz 500ms | Warning | Anklets |
| FallDetected | Extended buzz 1000ms | Critical | All nodes |
| FastObjectDetected | Rapid pulses (50ms × 5) | Critical | Pendant, eyewear |

### Directional Haptic Routing

**Directional haptics** provide spatially-relevant alerts:

| Threat Direction | Primary Haptic | Secondary Haptic |
|------------------|----------------|------------------|
| Front | Pendant piezo | Both bracelets (light) |
| Front-left | Left bracelet (strong) | Pendant (light) |
| Left | Left bracelet (strong) | Anklet left (light) |
| Rear-left | Left anklet (strong) | Left bracelet (light) |
| Rear | Both anklets | Both bracelets (light) |
| Rear-right | Right anklet (strong) | Right bracelet (light) |
| Right | Right bracelet (strong) | Anklet right (light) |
| Front-right | Right bracelet (strong) | Pendant (light) |

---

## Gait Analysis

The anklet IMUs are the primary input for gait analysis:
- Step frequency (cadence)
- Step regularity (gait coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right comparison)
- Pre-stumble signature detection

These feed PentaTrack's `WearerSelfMotion` anomaly detection and generate predictive stumble alerts before falls occur.

**Gait event transmission:** Anklets perform on-node gait event detection, transmitting events (not raw IMU data) to belt node. Raw IMU bursts sent only on anomaly detection for detailed analysis.

---

## Civilian Transfer Applications

- Fall detection and gait analysis for elderly or mobility-impaired individuals
- Sports performance analytics (stride, cadence, impact forces)
- Physical therapy monitoring and progress tracking
- Industrial worker safety (proximity alerts in factories and construction)
- Personal security for vulnerable populations (journalists, aid workers)
- Accessibility: haptic spatial awareness for visually impaired users
- Research platform for distributed wearable sensing architecture

---

## Firmware

Embedded firmware (`firmware/`) runs on `no_std` Rust targeting ARM Cortex-M (STM32, nRF5340, i.MX RT). Each node type has a dedicated binary. Shared logic (drivers, BAN protocol) in `firmware/src/lib.rs`.

**Key firmware responsibilities:**
- Sensor drivers (OMNI-SENSE abstraction)
- On-node processing (event detection, compression)
- BAN protocol (BLE/UWB communication to belt node)
- Power management
- Local storage (SD card if configured)
- **NO external network connectivity** (firmware for non-belt nodes has no WiFi/cellular drivers)

See `firmware/firmware.md` for the full firmware architecture specification.

---

## Hardware

Reference PCB designs for all node variants:
- `hardware/schematic/pendant_node/` — Standard and medallion variants
- `hardware/schematic/360_pendant_node/` — 360° curved pendant (separate node type)
- `hardware/schematic/bracelet_node/` — Single design, multiple configurations
- `hardware/schematic/belt_node/` — MCU, Linux SoM, high-performance variants
- `hardware/schematic/anklet_node/` — Single design, multiple configurations
- `hardware/schematic/eyewear_node/` — Clip-on and frame-integrated variants
- `hardware/testing/test_jig_pcb/` — Production test jig
- `hardware/hardware_config.md` — Interface standards, pin maps, power architecture

---

## Mechanical

Enclosure designs for all node types:
- `mechanical/cad/` — STEP files for all enclosures
- `mechanical/stl/` — STL files for 3D printing
- `mechanical/mechanical_spec.md` — Enclosure specifications

**Materials requirements:**
- Skin-contact surfaces: Grade 5 titanium, 316L surgical steel, or medical-grade silicone
- IP rating: IP67 for bracelets, anklets, pendant (water exposure)
- IP54 for belt node (splashes, not submersion)
- EN 1811:2011 nickel release compliance

---

## Roadmap

- **Phase 1.** Belt-node-only bench prototype — full sensor stack, body-frame fusion of one node, IMU-driven drift correction.
- **Phase 2.** Add bracelet + pendant — demonstrate multi-node body-frame fusion and cross-node drift correction.
- **Phase 3.** Full mesh of all six nodes with BAN protocol. Walk-through calibration. Sparse world model complete.
- **Phase 4.** 360° curved pendant prototype. Camera array stitching. Dense SLAM integration. Extreme velocity detection integration.
- **Phase 5.** Comfort, durability, weight, water resistance, hypoallergenic-material, and human-factors study. Thermal management validation.
- **Phase 6.** Companion app — both world model views operational. Full recording and legal export. Remote access via cellular.
- **Phase 7.** Public open-data release of body-frame trajectory dataset for the research community.
- **Parallel Track.** Extreme-velocity sensing physics characterization and production hardening.

---

## Getting Started

```bash
git clone https://github.com/ungatedminds/sentinel-wear
cd sentinel-wear
cargo build --workspace
cargo test --workspace

# Run belt controller in simulation mode
cargo run -p sentinel-belt-controller -- --simulated --config scenarios/sim_basic.toml

# Build firmware for pendant node
cd firmware
cargo build --target thumbv7em-none-eabihf --bin pendant_node --release

# Build firmware for 360° pendant
cargo build --target thumbv7em-none-eabihf --bin pendant_360 --release

# Build firmware for belt node (Linux SoM variant)
cargo build --target aarch64-unknown-linux-gnu --bin belt_node --release
```

---

## Disclaimer

SENTINEL-WEAR is a research and education project. It is not a medical device, not certified personal protective equipment, not a self-defense product, and not a substitute for any law-enforcement, medical, or emergency service. The maintainers make no warranty of fitness for any safety-critical use. Use at your own risk for educational purposes only.

**Extreme velocity detection:** The system can detect fast-moving objects within millisecond timeframes. This detection capability does not imply any ability to prevent impact from such objects. Physical interception of projectiles is explicitly out of scope.

---

## License

MIT for code. CERN-OHL-S v2 for hardware. CC BY 4.0 for documentation.
