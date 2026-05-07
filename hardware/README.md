# Hardware — SENTINEL-WEAR

**Reference designs for distributed wearable sensing nodes in jewelry form factor.**

**Version:** 0.3 | **Status:** Research Reference Design
**License:** CERN-OHL-S v2 for hardware designs

---

## 1. Overview

This directory contains the hardware reference designs for SENTINEL-WEAR's sensing nodes: pendant, bracelet, belt, anklet, and (optional) eyewear form factors. Each node carries some subset of mmWave radar, IMU, microphone array, short-range LiDAR/ToF, environmental sensors, BAN radio, battery, and haptic actuator as appropriate to its form factor.

**Scope:** Sensing-only effectors. No kinetic, projectile, energetic, pyrotechnic, or chemical effectors. No active protection system actuators. See `legal/compliance.md` and `legal/research_ethics.md` for complete scope definition.

---

## 2. Architectural Foundation — The Sole Gateway Principle

### 2.1 Critical Architectural Constraint

**Only the Belt Node connects to external networks.** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via Body-Area Network (BAN) to the belt node.** They have no WiFi, no cellular, no direct connection to the companion app, and no external network interface of any kind.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SENTINEL-WEAR Hardware Connectivity Model               │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  ⚠️  BELT NODE:     WiFi + Cellular + BLE (to external devices)             │
│  ⚠️  ALL OTHER NODES: BAN only (BLE/UWB to belt, no external connectivity)   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Why This Architecture?

| Constraint | WiFi on Wearable | BLE on Wearable | UWB on Wearable |
|------------|------------------|-----------------|-----------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Form factor impact** | Requires larger battery, antenna space | Minimal | Moderate |
| **Wearability** | Breaks jewelry form factor | Preserves jewelry form | Compatible |
| **Determinism** | Variable latency, contention | Consistent, schedulable | Excellent timing |
| **Antenna size** | Requires substantial antenna clearance | Very small antenna | Small antenna |
| **Battery impact** | Hours of runtime | Days of runtime | Hours if continuous |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power, thermal, and size requirements. BLE and UWB are the only viable BAN technologies for jewelry-scale wearables. All external connectivity (WiFi, Cellular) is concentrated at the belt node where larger form factor and battery are acceptable.

### 2.3 Comparison to Multi-Radio BLE Architecture

Some systems consider multiple BLE radios on a single hub for parallel connections. For SENTINEL-WEAR, this is **NOT recommended** because:

| Factor | Single BLE Radio | Multiple BLE Radios |
|--------|------------------|---------------------|
| RF self-interference | None (single transmitter) | High risk (radios centimeters apart) |
| Antenna coupling | Not applicable | Significant, causes receiver desensitization |
| Power consumption | 5-15 mW | 20-60 mW (multiple active radios) |
| Firmware complexity | Manageable | High (coordination required) |
| PCB complexity | Standard | High (isolation required) |
| Performance gain | — | Marginal at wearable scale |

**The correct solution for improved BLE performance is antenna diversity, not multi-radio.** See Section 7 for antenna diversity architecture.

### 2.4 Regulatory Compliance

All nodes comply with:
- **FCC Part 15** (unlicensed RF operation in ISM bands)
- **CE RED** (EU Radio Equipment Directive)
- **ISED** (Canada) equivalent requirements
- **UL/IEC 60950-1** or **IEC 62368-1** for electrical safety

Cellular modules (belt node only) require carrier certification per specific module.

---

## 3. Transport Layer Hierarchy

### 3.1 Bandwidth Hierarchy

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

### 3.2 UWB Bandwidth Capabilities — Detailed

UWB supports continuous streaming at appropriate resolutions:

| Stream Type | Bandwidth Required | UWB Capability | Notes |
|-------------|-------------------|----------------|-------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled | Instant baseline for 360° |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled | Standard quality for all cameras |
| 8 cameras × 720p H.264 | ~8-15 Mbps | ❌ Exceeds sustained | Requires WiFi or burst mode |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled | Good quality panorama |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ✅ At max | High quality panorama |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Exceeds | Requires WiFi |
| Single camera 720p-1080p | ~1-3 Mbps | ✅ Handled | Forward camera stream |

### 3.3 Tiered Progressive Quality Strategy

When bandwidth is constrained, the system can start with low-resolution baseline and progressively improve:

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

**This is a capability, not a limitation.** The architecture supports graceful degradation and progressive improvement.

---

## 4. Sensing-Only Scope

The hardware in this directory is **sensing-only**. No design in this directory implements:

- Active emitters above eye-safe educational classes.
- RF emitters operating outside FCC Part 15 / CE RED unregulated bands.
- Mechanical actuators capable of acting against any external object.
- Energy storage at densities or chemistries beyond standard wearable lithium-polymer.
- Any mechanism intended to provide protective function in the personal-protective-equipment sense.
- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

Researchers contributing hardware designs are asked to confirm in their pull requests that the contribution stays within these bounds. PRs proposing active-response hardware will be closed and the contributor referred to `docs/theory/future_research.md` for the appropriate research-domain context.

See `legal/compliance.md`, `legal/research_ethics.md`, and `legal/export_control_posture.md` for the project's full posture.

---

## 5. Node Types and Hardware Variants

### 5.1 Principle: Hardware Variants vs Configuration Options

**Hardware Variant:** Requires different PCB design, physical form factor, or core component selection. Documented as separate design in `schematic/` subdirectory.

**Configuration Option:** Same PCB design, different component population or firmware settings. Documented as options within a variant's README.

This distinction prevents variant proliferation while acknowledging genuinely different designs.

---

## 6. Node Type 1: Pendant / Necklace Node

The highest-information-value node. Positioned at chest level for upper-hemisphere coverage.

### 6.1 Hardware Variants

| Variant | Form Factor | Key Differentiators | Directory |
|---------|-------------|---------------------|-----------|
| **A — Standard Flat** | Flat medallion, 50×50×15mm max | mmWave + IMU + acoustic, optional camera | `schematic/pendant_node/` |
| **B — 360° Curved** | Curved arc pendant, 120-160mm arc | 4-8 camera array, 360° coverage | `schematic/360_pendant_node/` |
| **C — Medallion** | Larger circular, 60mm diameter × 20-25mm | Higher compute, higher-capacity battery | `schematic/pendant_node/` |
| **D — Tactical** | 70-80mm profile | Extended battery (up to 2000 mAh), belt power input | `schematic/pendant_node/` |
| **E — Event-Enhanced** | Based on B or C | Event cameras added for extreme velocity detection | `schematic/360_pendant_node/` |

### 6.2 Configuration Options (Within Variants)

**For Variants A, C, D:**

| Option | Values | Implementation |
|--------|--------|----------------|
| Camera | None / Forward / Wide-angle | Footprint on PCB, optionally populated |
| UWB | Disabled / Enabled | Footprint on PCB, optionally populated |
| Battery capacity | 150 / 250 / 400 mAh | Different cell in same enclosure |
| Microphone count | 3 / 4 / 6 element | Different population |

**For Variant B (360° Curved):**

| Option | Values | Implementation |
|--------|--------|----------------|
| Camera count | 4 / 6 / 8 | Different population on flex PCB |
| Camera resolution | QVGA / VGA / 720p / 1080p | Same sensors, different mode |
| Stitching location | Pendant / Belt | Vision processor placement |
| UWB | Disabled / Enabled | For bandwidth augmentation |

### 6.3 Connectivity

**All pendant variants:**
- BLE 5.x (mandatory) — control plane, metadata
- UWB (optional) — timing, ranging, moderate-bandwidth streaming
- **NO WiFi** — not present on any pendant variant
- **NO Cellular** — not present on any pendant variant
- All data routes via BAN to belt node → belt relays to companion app

### 6.4 360° Curved Pendant Architecture

The 360° pendant (Variant B) uses a **wired internal camera bus** architecture:

```
4-8 Cameras → MIPI CSI-2 (internal) → Vision Processor on Pendant
                                              │
                                              ├── Stitch on-pendant → Single 360° stream
                                              └── Encode (H.264/H.265)
                                                      │
                                                      ▼
                                                  UWB to Belt
                                                      │
                                                      ▼
                                              Belt → WiFi → Companion App
```

**Hardware synchronization:** All cameras share a single FSYNC line for simultaneous capture. PCB trace matching ensures < 10 ns skew between cameras.

**Bandwidth over UWB:** Supports 360° at 2K-2.5K resolution sustained. 4K requires WiFi (not available on pendant) or belt-assisted progressive quality strategies.

**Power and Thermal:**

| Mode | Power Draw | Runtime (800 mAh) |
|------|------------|-------------------|
| Sparse sensing only (no cameras) | ~500 mW | ~6 hours |
| Cameras active, VGA quality | ~2.5 W | ~1.2 hours |
| Cameras active, 720p quality | ~4 W | ~0.7 hours |
| With belt power input | Unlimited (belt-sourced) | N/A |

Thermal management required for 2-4W sustained operation:
- Thermal vias under vision processor
- Air gap between PCB and enclosure
- Maximum pendant surface temperature: 45°C (air side), 42°C (skin side)
- Duty cycling to reduce sustained thermal load

---

## 7. Node Type 2: Bracelet Node (×2)

Forearm-hemisphere sensing and directional haptic output. Worn on both wrists.

### 7.1 Hardware Design

**Single PCB design** supports all configurations. Different options achieved through component population and firmware configuration.

| Component | Options |
|-----------|---------|
| MCU | nRF5340, STM32WB55, Silicon Labs EFR32BG24 |
| mmWave Radar | Acconeer XR112 (60 GHz, ultra-compact) — recommended |
| IMU | BMI270 or ICM-42688-P |
| Haptic | TI DRV2605L driver + LRA actuator |
| Optional Camera | OV2640, HM01B0 (QVGA), or IMX219 |
| Optional UWB | Qorvo DW3000 footprint |
| Battery | 150-400 mAh (configurable) |

### 7.2 Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Standard / Extended | Different component population |
| Camera | None / Forward-facing | Optional module population |
| UWB | Disabled / Enabled | Optional module population |
| Battery | 150 / 250 / 400 mAh | Different cell sizes |

### 7.3 Connectivity

- BLE 5.x (mandatory)
- UWB (optional)
- **NO WiFi**
- **NO Cellular**
- Data routes via BAN to belt node only

### 7.4 Antenna Orientation

mmWave radar antenna must face outward (dorsal side of wrist) because body absorption significantly attenuates signals facing the wrist. Design places antenna on outer surface of bracelet case.

---

## 8. Node Type 3: Belt Node — Primary Hub and Sole External Gateway

The central coordination unit. All other nodes communicate only with the belt. The belt is the **only node with external network connectivity**.

### 8.1 Role

| Function | Belt Node Responsibility |
|----------|--------------------------|
| Compute hub | Runs fusion algorithms, PentaTrack, SLAM |
| Torso reference | IMU defines body-frame origin (+Y forward, +X right, +Z up) |
| BAN hub | Routes all inter-node communication |
| **External gateway** | **ONLY node with WiFi and Cellular** |
| App server | Embedded HTTP/WebSocket server for companion app |
| Recording manager | Aggregates recordings from all nodes |
| Media relay | Receives BAN streams, relays via WiFi/Cellular to app |

### 8.2 Hardware Variants

| Variant | Compute Platform | Capability | Battery | Runtime |
|---------|------------------|------------|---------|---------|
| **A — MCU** | STM32H7 (Cortex-M7, 480 MHz) | Sparse tracking only | ~2000 mAh | 15-20 hrs (sparse) |
| **B — Linux SoM** | Raspberry Pi CM4 / NXP i.MX 8M Plus | Full SLAM, streaming | 5000-7000 mAh | 10-15 hrs (sparse), 5-8 hrs (active) |
| **C — High-Performance** | Qualcomm SA8155P or NXP i.MX 8M Plus with NPU | GPU-class SLAM, neural stitching | 5000-7000 mAh | 5-8 hrs (active) |
| **D — ESP32** | ESP32-S3 | Minimal deployment, 2-3 nodes | ~2000 mAh | 12-15 hrs |
| **E — Hot-Swappable** | Based on B or C | Dual battery bays, swap without downtime | 2× 5000 mAh | Unlimited with swaps |

### 8.3 Connectivity (Belt Node Only)

| Interface | Purpose | Belt Only? |
|-----------|---------|------------|
| WiFi 802.11ac/ax | Primary companion app connection | **Yes** |
| Cellular (LTE/5G) | Remote access, emergency alerts | **Yes** |
| Bluetooth 5.3 | Local direct app connection (fallback) | **Yes** |
| USB-C | PC companion app, charging | **Yes** |
| BLE 5.x | BAN to other nodes | All nodes |
| UWB (optional) | BAN precision timing | Belt + optional on other nodes |

### 8.4 Cellular Modules Supported

| Module | Technology | Speed | Use Case |
|--------|------------|-------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | Alerts, metadata |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | Compressed video |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12, eSIM | 300+ Mbps | Multi-carrier |

### 8.5 Power Budget

| Mode | Active Functions | Power Draw | Runtime (5000 mAh) |
|------|------------------|------------|-------------------|
| Minimal | BAN hub, sparse tracking | ~500 mW | ~37 hours |
| Standard | BAN, sparse tracking, WiFi idle | ~1.4 W | ~13 hours |
| Full Active | BAN, dense SLAM, 360° stitching, streaming | ~4.5 W | ~4 hours |
| Remote Streaming | Full Active + cellular | ~5.3 W | ~3.5 hours |

### 8.6 Thermal Management (Linux SoM Variants)

Sustained compute generates 3-7 W. Requirements:
- Thermal pad under SoM
- Vented or heat-spreader enclosure
- NTC thermistor for monitoring
- Firmware throttles SLAM frame rate above 40°C
- Maximum skin-contact temperature: 42°C

### 8.7 Antenna Diversity (Recommended Configuration)

**Single BLE radio + Dual antennas = Improved reliability**

```
Belt Node BLE Radio
    │
    ├── Antenna Switch (SPDT, <1 µs switch time)
    │       ├── Left-side antenna (toward wearer's left)
    │       └── Right-side antenna (toward wearer's right)
    │
    └── Radio dynamically selects antenna with better RSSI
```

**Why antenna diversity:**
- Human body blocks 2.4 GHz signals (10-30 dB attenuation through torso)
- Arm and body orientation constantly change
- Dual antennas provide spatial diversity
- NO self-interference (only one transmitter active at a time)

**Benefits:**
- Reduced packet loss in body-shadowing conditions
- More deterministic latency
- Critical for extreme velocity detection reliability

**Configuration:**

```toml
[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"          # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5          # Switch if other antenna is 5 dB better
preferred_antenna = "auto"       # "left" | "right" | "auto"
```

---

## 9. Node Type 4: Anklet Node (×2)

Ground-plane sensing, gait analysis, lower-hemisphere coverage.

### 9.1 Hardware Design

**Single PCB design** supports all configurations.

| Component | Options |
|-----------|---------|
| MCU | Nordic nRF5340 or Silicon Labs EFR32BG24 |
| Range sensor | VL53L5CX ToF (8×8, 6 m) or short-range LiDAR |
| IMU | BMI270 (high dynamic range for heel-strike: 5-10 g) |
| Haptic | TI DRV2605L + LRA |
| Optional mmWave | Acconeer XR112 |
| Optional Camera | HM01B0 (QVGA) or OV2640 |
| Optional UWB | DW3000 footprint — **recommended for gait synchronization** |
| Battery | 300-500 mAh |

### 9.2 Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal (ToF only) / Extended (ToF + LiDAR + mmWave) | |
| Camera | None / Downward-facing | Ground obstacles |
| UWB | Disabled / Enabled | **Recommended for gait sync** |
| Battery | 300 / 500 mAh | |

### 9.3 Gait Analysis Capabilities

On-node gait event detection transmits events (not raw IMU) to belt:
- Step frequency (cadence)
- Step regularity (coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right)
- Pre-stumble signature detection
- Gait phase: stance / swing / strike / push at ≥ 200 Hz

Raw IMU burst sent only on anomaly detection for detailed analysis.

### 9.4 Connectivity

- BLE 5.x (mandatory)
- UWB (optional, **recommended**)
- **NO WiFi**
- **NO Cellular**

---

## 10. Node Type 5: Eyewear Node (Optional)

Head-stabilized forward-hemisphere sensing, fast transient detection, SLAM visual odometry anchor.

### 10.1 Hardware Variants

| Variant | Form Factor | Sensors | Weight | Battery |
|---------|-------------|---------|--------|---------|
| **A — Event-Only Clip-On** | 30×20×8 mm clip | Event camera + IMU | < 8 g | 50-80 mAh |
| **B — Event + Camera Clip-On** | Same as A | Event + conventional camera + IMU | < 10 g | 50-80 mAh |
| **C — Frame-Integrated** | Temple arm segments | Event + forward + side cameras + IMU | 20-40 g | 80-150 mAh |
| **D — Headband** | Forehead band | Full camera array + IMU | 30-50 g | 80-150 mAh or belt cable |

### 10.2 Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor type | Event-only / Event + Camera | |
| Camera count | 1 / 2 / 3 | Forward + sides |
| UWB | Disabled / Enabled | For head-torso sync |
| Power source | Battery / Belt cable (Variant C/D) | Cable for extended runtime |

### 10.3 SLAM Integration

- **Variant A:** Sparse optical flow only, limited SLAM contribution
- **Variant B:** Conventional camera provides keyframes for loop closure
- **Variant C/D:** Wide-baseline stereo between forward and side cameras enables depth estimation; 180°+ coverage prevents visual odometry failure during rotation

### 10.4 Connectivity

- BLE 5.x (mandatory)
- UWB (optional)
- **NO WiFi**
- **NO Cellular**

---

## 11. Body-Area Network (BAN) Architecture

### 11.1 BLE Scheduling and Determinism

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
├── Slot 25-28 ms: RESERVED (extreme velocity / critical alerts)
└── Slot 28-30 ms: Reserved (retransmits)
```

### 11.2 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### 11.3 Node Priority for Bandwidth Allocation

When multiple nodes have data to transmit simultaneously, the belt node serves higher-priority nodes first:

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

**Override:** Critical alerts (fall detected, extreme velocity) are transmitted immediately regardless of priority slot.

### 11.4 QoS Classes

| QoS Class | Priority | Traffic Types | Max Latency |
|-----------|----------|---------------|-------------|
| Critical | Highest | Extreme velocity, fall detection, emergency alerts | < 5 ms |
| High | Second | Gait anomaly, stumble precursor | < 20 ms |
| Medium | Third | Presence detection, tracking update | < 50 ms |
| Low | Fourth | Battery status, health telemetry | < 500 ms |

**Critical QoS implementation:**
- Preempts all other traffic
- Uses reserved slot (25-28 ms) if needed
- Bypasses normal scheduling
- Essential for extreme velocity detection (< 5 ms latency)

### 11.5 Dynamic Scheduling

Context-aware adjustments improve responsiveness:

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
extreme_velocity_preempt = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
```

### 11.6 UWB Role Configuration

UWB can be enabled on any node with UWB hardware (DW3000 class). Roles:

**Role: `timing_only`**
- Sub-nanosecond time synchronization between nodes
- cm-level ranging for body-frame reconstruction
- Minimal data transfer
- Lowest power UWB mode (~50 mW)

**Role: `timing_and_ranging`**
- All timing capabilities
- Distance measurement between nodes for body-frame accuracy
- Occasional data bursts
- Moderate power (~80 mW)

**Role: `full_bandwidth`**
- All timing capabilities
- Sustained streaming up to 5-6 Mbps
- Single camera streaming, 360° at 2K resolution
- Progressive quality strategies
- Higher power (100-150 mW sustained)

**Configuration:**

```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"     # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false      # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]
```

---

## 12. RF Coexistence

### 12.1 Frequency Bands in Use

| Technology | Frequency Band | Potential Conflicts |
|------------|---------------|---------------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz, 5 GHz | BLE at 2.4 GHz |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave Radar | 60 GHz | None (separate band) |
| Cellular | Various (700 MHz - 3.5 GHz, 24-47 GHz 5G mmWave) | Minor at adjacent bands |

### 12.2 Coexistence Strategies

1. **WiFi prefers 5 GHz band** — eliminates 2.4 GHz conflict with BLE
2. **Antenna separation on belt** — Cellular/WiFi antennas on exterior, BLE/UWB on interior toward body
3. **Time-division multiplexing** — If 2.4 GHz WiFi must be used, coordinate timing with BLE
4. **UWB is in separate band** — No conflict with BLE or WiFi

### 12.3 Antenna Placement Strategy (Belt Node)

```
Belt Node Enclosure (Top View)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   [Cellular Antenna]                    [WiFi Antenna]          │
│   (Exterior, separate from body)        (Exterior)              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              PCB / Internal Electronics             │      │
│   │                                                     │      │
│   │   [BLE Antenna]         [BLE Antenna]               │      │
│   │   (Interior, toward body for BAN)                   │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Principle:**
- Cellular/WiFi antennas on exterior of belt (better propagation, away from body)
- BLE antennas on interior (body is the communication medium for BAN)
- Physical separation between 2.4 GHz antennas reduces coupling

### 12.4 Configuration

```toml
[rf_coexistence]
wifi_prefer_5ghz = true        # Strongly recommended
ble_wifi_timeshare = false     # Set true only if 2.4 GHz WiFi required
ble_gap_during_wifi_ms = 5     # Gap between BLE and WiFi if sharing
```

---

## 13. Extreme Velocity Detection — Hardware Considerations

### 13.1 Doppler Radar Configuration

For production projectile detection (rifle velocity ~850 m/s):

| Requirement | Value |
|-------------|-------|
| Doppler velocity range | ±300 m/s or higher |
| Update rate | ≥ 10,000 Hz (10 kHz) |
| Latency | < 100 microseconds to detection |
| Tradeoff | Reduced range resolution acceptable |

**Modules supporting extended Doppler:**
- TI IWR6843 (with custom chirp configuration) — recommended
- Infineon BGT60ATR24C — research needed
- Acconeer XR112 — research needed

### 13.2 Event Camera Requirements

| Requirement | Value |
|-------------|-------|
| Latency | < 1 millisecond to detection |
| Resolution | QVGA minimum (higher not required) |
| Frame rate | Event-based (no fixed frame rate) |

### 13.3 Latency Budget

| Stage | Latency | Notes |
|-------|---------|-------|
| Doppler detection | 10-100 µs | Hardware event |
| Event camera trigger | < 1 ms | Fast transient detection |
| MCU processing | 100-500 µs | Detection classification |
| BLE queue (worst case) | 0-30 ms | Depends on schedule slot |
| BLE queue (Critical QoS) | 0-5 ms | Reserved slot, preemption |
| BLE transmission | 1-3 ms | Packet airtime |
| Belt reception | < 1 ms | Processing |
| Alert decision | 100-500 µs | Classification |
| Haptic trigger | < 1 ms | Actuator response |
| **Total (standard)** | **~5-35 ms** | Depends on scheduling |
| **Total (Critical QoS)** | **~5-10 ms** | Deterministic |

### 13.4 Firmware Configuration

```toml
[extreme_velocity]
enabled = false                 # Opt-in feature
mode = "production"             # "disabled" | "research" | "production"
doppler_update_rate_hz = 10000  # High update rate for fast transients
event_camera_always_on = true
min_velocity_ms = 50            # Ignore objects below this velocity
max_detection_distance_m = 10
alert_on_detection = true
processing_priority = "realtime" # Preempts other processing
qos_class = "critical"          # Uses reserved slot, preempts other traffic
```

---

## 14. Power Architecture

### 14.1 Power Sources

| Source | Nodes | Notes |
|--------|-------|-------|
| LiPo battery | All nodes | Standard wearable chemistry |
| USB-C charging | All nodes | 5V input |
| Belt power input | Pendant (tactical), Eyewear (frame-integrated) | Extended runtime |
| Qi wireless | Anklet, Bracelet | Optional |

### 14.2 Power Domains

| Domain | Voltage | Usage |
|--------|---------|-------|
| V_BAT | 3.7-4.2V | Radio PA, haptic peak |
| V_SYS | 3.3V | MCU, sensors |
| V_RADIO | 3.3V | BLE/UWB radio |
| V_SENSOR | 3.3V / 1.8V | High-power sensors |
| V_CAM | 3.3V / 2.8V | Camera rail (gated) |

### 14.3 Battery Safety

All LiPo cells: UL 1642 or IEC 62133 certified
- Overcharge cutoff: 4.25V
- Overdischarge cutoff: 3.0V
- Thermal cutoff: NTC thermistor, > 60°C during charge
- Short-circuit: PTC fuse or protection IC

### 14.4 Power Estimates by Node

| Node Variant | Sleep (mW) | Active (mW) | Peak (mW) |
|--------------|------------|-------------|-----------|
| Pendant Standard | 10-50 | 200-500 | 800 |
| Pendant 360° | 50-100 | 2500-4500 | 6000 |
| Pendant Medallion | 20-60 | 300-800 | 1200 |
| Bracelet Minimal | 5-15 | 100-250 | 400 |
| Bracelet Standard | 5-20 | 100-300 | 500 |
| Bracelet Extended | 10-30 | 200-500 | 700 |
| Belt MCU | 20-50 | 300-800 | 2000 |
| Belt Linux SoM | 100-300 | 1000-5000 | 8000 |
| Anklet Minimal | 5-15 | 100-250 | 400 |
| Anklet Extended | 10-25 | 150-400 | 600 |
| Eyewear Clip-On | 5-10 | 100-300 | 500 |
| Eyewear Frame | 10-30 | 200-600 | 1000 |

### 14.5 Runtime Estimates

| Node | Battery | Standard Mode | Full Active |
|------|---------|---------------|-------------|
| Pendant Standard | 250 mAh | 24-48 hours | N/A |
| Pendant 360° | 800 mAh | 4-6 hours | 1-2 hours |
| Bracelet Standard | 250 mAh | 24-48 hours | 12-24 hours (with camera) |
| Belt MCU | 2000 mAh | 15-20 hours | N/A |
| Belt Linux SoM | 5000 mAh | 10-15 hours | 4-8 hours |
| Belt Hot-Swappable | 2× 5000 mAh | Unlimited (with swaps) | Unlimited |
| Anklet Standard | 300 mAh | 48-72 hours | 24-48 hours |
| Eyewear Clip-On | 60 mAh | 8-16 hours | N/A |

---

## 15. Materials and Human Factors

### 15.1 Hypoallergenic Materials (Skin Contact)

All skin-contact surfaces comply with EN 1811:2011 nickel release requirements:

**Acceptable:**
- Grade 5 titanium (Ti-6Al-4V)
- 316L surgical stainless steel
- Medical-grade silicone (ISO 10993)
- Hard-coat anodized aluminum (enclosures only, coating intact)

**Not Acceptable for Skin Contact:**
- 6061/7075 aluminum alloys (bare)
- Unplated brass
- Zinc die-cast
- Non-surgical stainless steel

### 15.2 Water Resistance

| Node | Minimum Rating | Test Method |
|------|----------------|-------------|
| Pendant | IP67 | 1 m submersion, 30 min |
| Bracelets | IP67 | 1 m submersion, 30 min |
| Belt Node | IP54 | Splash test |
| Anklets | IP67 | 1 m submersion, 30 min |
| Eyewear Clip-On | IPX4 | Sweat/splash |
| Eyewear Frame | IPX4 | Sweat/splash |

### 15.3 Weight Targets

| Node | Target Weight | Notes |
|------|---------------|-------|
| Pendant Standard | 30-60 g | Comfortable for extended wear |
| Pendant 360° | 50-120 g | Heavier due to cameras/battery |
| Pendant Medallion | 80-150 g | Premium, larger profile |
| Bracelet | 25-60 g | Watch-like |
| Belt MCU | 100-150 g | Including battery |
| Belt Linux SoM | 200-300 g | Larger battery bank |
| Anklet | 40-70 g | Slim profile |
| Eyewear Clip-On | < 8 g | Must not burden glasses |
| Eyewear Frame | 20-40 g | Integrated electronics |

### 15.4 Thermal Comfort

Maximum skin-contact temperature: 42°C (acceptable), 40°C (target)

Thermal management required for:
- 360° pendant during active streaming
- Belt Linux SoM during SLAM + streaming
- Bracelets/anklets during sustained haptic feedback

Mitigation strategies:
- Thermal vias under heat-generating ICs
- Air gap between PCB and enclosure
- Duty cycling for sustained loads
- Enclosure material selection (metal for heat spreading, plastic for insulation)

---

## 16. Camera Privacy Controls — Available Options

All camera privacy controls are user-selected. None are mandatory by the architecture.

### 16.1 Option A: Hardware Power Switch (User-Installed)

- Physical switch cuts V_CAM power rail
- When DISABLED: camera is physically unpowered. No software override.
- PCB provides footprint for this switch on all camera-capable nodes
- MOSFET on the V_CAM rail; switch controls gate drive
- Firmware reads `CAM_HW_SW_GPIO` at boot if pin is populated
- Users who want unconditional physical camera disable select this option

### 16.2 Option B: Software-Only Control

- MCU controls V_CAM via MOSFET GPIO without physical switch
- Software scheduling, detection-triggered activation, or manual control
- Simpler hardware (no switch component)
- Users who prefer flexibility over guaranteed physical disable select this option

### 16.3 Option C: Always-On Camera

- Camera powered continuously; no gating
- Data handling (storage, streaming, retention) configured in `sentinel-wear.toml`
- Useful for continuous recording deployments

### 16.4 Configuration

```toml
[privacy.camera]
hardware_switch_installed = false   # Set true if physical switch is present
software_control_enabled = true
default_state = "disabled"          # "enabled" | "disabled"
schedule = []                        # Time ranges when enabled
trigger_on_detection = false         # Enable camera on detection event
```

---

## 17. Directory Structure

```
hardware/
├── schematic/
│   ├── pendant_node/           # Variants A, C, D (standard, medallion, tactical)
│   │   ├── variant_a_standard/
│   │   ├── variant_c_medallion/
│   │   ├── variant_d_tactical/
│   │   └── README.md
│   ├── 360_pendant_node/       # Variant B (360° curved) — separate node type
│   │   ├── rigid_flex_design/
│   │   ├── camera_array/
│   │   └── README.md
│   ├── bracelet_node/          # Single design, multiple configs
│   │   └── README.md
│   ├── belt_node/              # Variants A, B, C, D, E
│   │   ├── variant_a_mcu/
│   │   ├── variant_b_linux_som/
│   │   ├── variant_c_high_perf/
│   │   ├── variant_d_esp32/
│   │   ├── variant_e_hotswap/
│   │   └── README.md
│   ├── anklet_node/            # Single design, multiple configs
│   │   └── README.md
│   ├── eyewear_node/           # Variants A, B, C, D
│   │   ├── variant_a_clip_on/
│   │   ├── variant_b_clip_camera/
│   │   ├── variant_c_frame/
│   │   ├── variant_d_headband/
│   │   └── README.md
│   └── hardware_config.md      # Interface standards, pin maps
├── testing/
│   └── test_jig_pcb/           # Production test jig
│       └── README.md
├── bom/                        # Bill of materials templates
├── mechanical/                 # Link to mechanical specifications
└── README.md                   # This file
```

---

## 18. Testing and Quality Control

### 18.1 Test Jig

Universal Test Station (UTS) validates all node variants:

**Carrier board:**
- Test controller MCU (STM32H7 or RP2040)
- Host SBC (Raspberry Pi 5)
- Programmable power supply
- Current sensing (per rail)
- RF shield for radar isolation
- BAN golden node reference

**Node-specific test shields:**
- Pendant Shield: Radar reflector, acoustic speaker, camera target
- 360° Pendant Shield: Multi-position LED array, stitching target, sync analyzer
- Bracelet Shield: Radar absorber, haptic resonance fixture
- Belt Shield: WiFi AP, cellular simulator, thermal load fixture
- Anklet Shield: ToF target, IMU impact simulator
- Eyewear Shield: Event LED strobe, IMU gimbal

### 18.2 Mandatory Tests (All Nodes)

1. Power-on self-test
2. Rail verification (±5%)
3. Sensor enumeration
4. IMU gravity vector (9.81 ± 0.2 m/s² at rest)
5. BAN connectivity to belt node (< 200 ms)
6. Battery level verification
7. Firmware version match

### 18.3 Cellular Tests (Belt Node Only)

1. SIM detection
2. Module AT command response
3. Network registration (with live SIM)
4. Signal strength
5. Alert relay via cellular
6. Recording delivery to companion app

### 18.4 Antenna Diversity Tests (If Configured)

1. RSSI measurement on both antennas
2. Antenna switching verification
3. Body-shadowing test (cover one antenna, verify switch)
4. Latency measurement with diversity enabled vs disabled

### 18.5 IP67 Tests (Applicable Nodes)

1 m water depth, 30 minutes. Post-submersion: powers on, all sensors functional, rails within ±2%.

---

## 19. Component Selection Philosophy

### 19.1 No Pre-Constrained Specifications

Component lists in schematic READMEs are **recommendations and tested variants**, not locked specifications. Researchers may substitute:

- Different MCU families (ARM, RISC-V)
- Different sensor models with similar capabilities
- Different battery capacities
- Different radio modules (within regulatory compliance)

### 19.2 Requirements for Substitutions

- Same interface type (SPI, I2C, MIPI, etc.)
- Power within PCB design limits
- Pin-compatible or minor rework required
- Regulatory compliance maintained

---

## 20. Configuration Files

Hardware configuration exposed in `sentinel-wear.toml`:

```toml
[nodes]
pendant.type = "standard"           # "standard" | "360_curved" | "medallion" | "tactical" | "event_enhanced"
pendant.camera = false
pendant.uwb = false
pendant.battery_mah = 250

bracelets.sensor_set = "standard"    # "minimal" | "standard" | "extended"
bracelets.camera = false
bracelets.uwb = false

belt.variant = "linux_som"           # "mcu" | "linux_som" | "high_perf" | "esp32" | "hotswap"
belt.cellular = true
belt.cellular_module = "ec25"        # "ec21" | "ec25" | "rm502q" | "eg912y"
belt.battery_mah = 5000
belt.antenna_diversity = true        # Dual BLE antennas

anklets.sensor_set = "standard"
anklets.uwb = true                   # Recommended for gait sync

eyewear.variant = "clip_on"          # "clip_on" | "frame" | "headband"
eyewear.sensors = "event_only"       # "event_only" | "event_camera"

[ban]
primary = "ble"                      # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = true
uwb_role = "timing_and_ranging"      # "timing_only" | "timing_and_ranging" | "full_bandwidth"

[ban.scheduling]
mode = "dynamic"
extreme_velocity_preempt = true
reserved_slot_start_ms = 25

[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"

[connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
alert_via_cellular = true

[extreme_velocity]
enabled = false
mode = "production"
doppler_update_rate_hz = 10000
event_camera_always_on = true
qos_class = "critical"
```

---

## 21. Legal and Compliance

### 21.1 RF Compliance

- All BLE modules: FCC Part 15.247, CE RED Article 3.2
- All UWB modules: FCC Part 15.250, CE RED
- Cellular modules: Carrier certification required per module

### 21.2 Electrical Safety

- UL 60950-1 / IEC 62368-1
- Battery: UL 1642 / IEC 62133

### 21.3 EMC

- FCC Part 15 Subpart B (unintentional radiators)
- CISPR 32 Class B (residential)

### 21.4 Export Control

See `legal/export_control_posture.md` for complete analysis. Summary:
- mmWave radar (60 GHz): Generally not export-controlled
- UWB: Generally not export-controlled in most jurisdictions
- Standard electronics: Standard export classifications apply

---

## 22. References

- `hardware_config.md` — Interface standards, pin maps, power architecture
- `docs/theory/ban_bandwidth_budget.md` — BLE/UWB allocation strategies
- `docs/theory/power_management.md` — Battery and power optimization
- `docs/theory/thermal_management.md` — Thermal design for wearables
- `docs/theory/rf_coexistence.md` — RF interference mitigation
- `docs/theory/extreme_velocity_sensing.md` — Projectile detection physics
- `docs/theory/doppler_radar_config.md` — Extended Doppler configuration
- `firmware/firmware.md` — Firmware architecture per node type
- `mechanical/mechanical_spec.md` — Enclosure designs
- `legal/compliance.md` — Regulatory compliance details

---

## 23. Contributing

When submitting hardware designs:

1. Confirm sensing-only scope (no effectors)
2. Confirm regulatory compliance (RF, electrical, EMC)
3. Provide complete BOM with supplier part numbers
4. Include test procedure for validation
5. Document any configuration options supported

PRs will be reviewed for scope compliance, technical correctness, and documentation completeness.

---

**End of Hardware README**
