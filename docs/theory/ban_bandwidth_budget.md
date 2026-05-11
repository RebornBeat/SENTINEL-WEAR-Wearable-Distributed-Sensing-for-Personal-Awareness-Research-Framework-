```markdown
# Body-Area Network Bandwidth Budget — SENTINEL-WEAR

**Project:** SENTINEL-WEAR
**Domain:** BAN architecture, bandwidth allocation, scheduling, and RF coexistence
**Implementation:** `firmware/src/ban_radio/`, `crates/sentinel-ban-protocol/`
**Status:** Production Reference

---

## 1. Purpose

This document specifies the complete Body-Area Network (BAN) architecture for SENTINEL-WEAR, including:

- Bandwidth hierarchy across all transport layers (BLE, UWB, WiFi, Cellular)
- BLE scheduling and determinism for jewelry-form-factor wearables
- UWB role configuration for precision timing and bandwidth augmentation
- QoS classes for traffic prioritization
- Antenna diversity vs multi-radio tradeoffs
- RF coexistence strategies
- Data contracts between nodes
- Configuration reference

**Critical architectural constraint:** Only the Belt Node has WiFi and Cellular connectivity. All other nodes communicate exclusively via BAN (BLE and/or UWB) to the belt node.

---

## 2. Transport Layer Hierarchy

### 2.1 Overview

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

### 2.2 BLE 5.x Detailed Capabilities

| Parameter | Value | Notes |
|-----------|-------|-------|
| Maximum theoretical rate | 2 Mbps | BLE 5.x PHY |
| Practical sustained throughput | 500 Kbps - 1.5 Mbps | Depends on connection interval, environment |
| Latency (best case) | 3-10 ms | Minimum connection interval |
| Latency (typical) | 10-30 ms | Standard connection interval |
| Latency (worst case) | 30-100+ ms | Congested RF, retries |
| Power (active TX/RX) | 5-15 mW | Per radio |
| Power (sleep between events) | < 1 mW | Low power mode |

**BLE strengths:**
- Lowest power transport
- Mature ecosystem, widespread hardware support
- Excellent scheduling determinism
- Body-coupled propagation (works well for BAN)

**BLE limitations:**
- Cannot sustain high-bandwidth video streaming
- Susceptible to 2.4 GHz interference (WiFi, microwave, other BLE)
- Latency varies with connection interval and queue depth

### 2.3 UWB Detailed Capabilities

| Parameter | Value | Notes |
|-----------|-------|-------|
| Maximum theoretical rate | 6.8 Mbps | DW3000 class |
| Practical sustained throughput | 4-6 Mbps | Conservative thermal management |
| Peak burst rate | 6.8 Mbps | Short duration |
| Time synchronization precision | < 1 ns | Sub-nanosecond possible |
| Ranging precision | 1-10 cm | Depends on configuration |
| Latency (typical) | 1-5 ms | For data transfer |
| Power (active) | 50-150 mW | Significantly higher than BLE |
| Power (idle/timing only) | 10-30 mW | Ranging/sync mode |

**UWB strengths:**
- Excellent timing precision
- Separate frequency band (3.1-10.6 GHz) — no BLE/WiFi interference
- Higher bandwidth than BLE
- Precise ranging capability

**UWB limitations:**
- Higher power than BLE
- More complex hardware integration
- Less mature ecosystem than BLE
- Cannot match WiFi for high-bandwidth streaming

### 2.4 WiFi Detailed Capabilities

| Parameter | Value | Notes |
|-----------|-------|-------|
| Maximum rate (802.11ac) | 433 Mbps - 1.7 Gbps | Per spatial stream |
| Maximum rate (802.11ax) | 600 Mbps - 2.4 Gbps | Per spatial stream |
| Practical sustained | 50-300+ Mbps | Real-world conditions |
| Latency (best case) | 1-5 ms | Local network |
| Latency (typical) | 5-20 ms | Congested network |
| Power (active) | 300-500+ mW | Streaming |
| Power (idle) | 50-100 mW | Connected, minimal traffic |

**WiFi strengths:**
- Highest bandwidth
- Standard protocol, universal support
- No special hardware needed beyond standard SoM

**WiFi limitations:**
- High power consumption (unacceptable for jewelry nodes)
- 2.4 GHz band conflicts with BLE
- Variable latency (less deterministic)

### 2.5 Cellular Detailed Capabilities

| Module Class | Technology | Speed | Power (Active) | Use Case |
|--------------|------------|-------|----------------|----------|
| LTE Cat 1 | 4G LTE | 5-10 Mbps | 150-300 mA | Alerts, metadata |
| LTE Cat 4 | 4G LTE | 50-150 Mbps | 400-600 mA | Compressed clips |
| LTE Cat 12 | 4G LTE | 300-600 Mbps | 500-800 mA | High-quality streaming |
| 5G Sub-6 | 5G | 100-1000+ Mbps | 600-1200 mA | Live 360° streaming |
| 5G mmWave | 5G | 1000+ Mbps | 800-1500 mA | Ultra-high bandwidth |

**Cellular strengths:**
- Works anywhere with cellular coverage
- No WiFi infrastructure required
- Remote access capability

**Cellular limitations:**
- Highest power consumption
- Data plan costs
- Regulatory requirements (carrier certification)
- Latency varies with network conditions

---

## 3. Node Connectivity Summary

### 3.1 Connectivity Matrix

| Node Type | BLE | UWB | WiFi | Cellular | Notes |
|-----------|-----|-----|------|----------|-------|
| Pendant — Standard | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Pendant — 360° Curved | ✅ Mandatory | ✅ Recommended | ❌ None | ❌ None | UWB for bandwidth |
| Pendant — Medallion | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Pendant — Tactical | ✅ Mandatory | ✅ Recommended | ❌ None | ❌ None | Extended runtime |
| Pendant — Event-Enhanced | ✅ Mandatory | ✅ Recommended | ❌ None | ❌ None | Extreme velocity |
| Pendant — Long-Range | ✅ Mandatory | ✅ Recommended | ❌ None | ❌ None | Max detection range |
| Bracelet | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Anklet | ✅ Mandatory | ⚠️ Recommended | ❌ None | ❌ None | UWB for gait sync |
| Eyewear | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Belt | ✅ Mandatory | ⚠️ Optional | ✅ Mandatory | ⚠️ Optional | Sole external gateway |

### 3.2 Why Non-Belt Nodes Have No WiFi/Cellular

| Constraint | WiFi on Jewelry | Cellular on Jewelry |
|------------|-----------------|---------------------|
| Power consumption | 300-1000+ mW active | 150-1200+ mW active |
| Thermal impact | Significant heat in small enclosure | Significant heat |
| Antenna requirements | Requires clearance, affects form factor | Requires clearance, antenna size |
| Battery impact | 10-20× faster drain than BLE | 20-50× faster drain than BLE |
| Form factor | Breaks jewelry aesthetic | Breaks jewelry aesthetic |
| Regulatory | FCC/CE certification required | Carrier certification required |

**Conclusion:** WiFi and cellular are only viable on the belt node where larger form factor, larger battery, and thermal management are acceptable.

---

## 4. BLE Scheduling and Determinism

### 4.1 The Problem with Naive BLE Usage

If all nodes transmit simultaneously without coordination:

```
All nodes attempt TX at same time
    ↓
BLE collisions on shared channel
    ↓
Packet loss, retries
    ↓
Unpredictable latency (10-100+ ms)
    ↓
Degraded body-frame tracking
    ↓
Missed extreme velocity detection
```

### 4.2 Time-Slotted Scheduling Solution

**Principle:** Allocate deterministic time slots within each BLE connection interval for each node.

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
├── Slot 25-28 ms: Reserved (alerts, retransmits, eyewear)
└── Slot 28-30 ms: Reserved (critical alerts, extreme velocity)
```

**Guarantees:**
- Each node has a guaranteed transmission window every cycle
- Latency is bounded by connection interval + slot offset
- Collisions are eliminated by design
- Retries only occur within reserved slot

### 4.3 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture, extreme velocity |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

**Human timescale context:**

| Human Function | Latency Threshold | Compatible Intervals |
|----------------|-------------------|---------------------|
| Haptic perception | 10-20 ms | 7.5-15 ms |
| Motion perception | 20-50 ms | 7.5-30 ms |
| Gait phase timing | 10-30 ms | 7.5-30 ms |
| Visual awareness | 50-100 ms | 30-100 ms |

**Default recommendation:** 30 ms interval provides good balance of latency, determinism, and power.

### 4.4 Dynamic Scheduling

**Principle:** Adjust scheduling based on current context.

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
# Context-aware adjustments
walking_anklet_priority_boost = 1.5    # Anklet slots increase by 50%
threat_pendant_priority_boost = 2.0     # Pendant slots double during threat
idle_interval_multiplier = 2.0          # Intervals double when idle
streaming_shift_to_uwb = true           # Move heavy traffic to UWB
extreme_velocity_preempt = true         # Detection preempts all other slots
```

### 4.5 Node Priority for Bandwidth Allocation

When multiple nodes have data ready simultaneously, the belt node serves higher-priority nodes first:

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

**Override:** Critical alerts (fall detected, extreme velocity) are transmitted immediately regardless of priority slot.

---

## 5. QoS Classes for Traffic Prioritization

### 5.1 QoS Class Definitions

| QoS Class | Priority | Max Latency | Preemption | Use Cases |
|-----------|----------|-------------|------------|-----------|
| Critical | Highest | 5 ms | Yes | Extreme velocity detection, fall detection, emergency alerts |
| High | Second | 20 ms | No | Gait anomaly, stumble precursor, threat detection |
| Medium | Third | 50 ms | No | Presence detection, tracking update, position data |
| Low | Fourth | 500 ms | No | Battery status, health telemetry, diagnostics |

### 5.2 QoS Implementation

**Critical QoS behavior:**
- Bypasses normal transmission queue
- Can preempt ongoing transmission (with coordination)
- Reserved time slot guaranteed
- Immediate transmission when ready

**Configuration:**

```toml
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
reserved_slot_enabled = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor", "threat_detection"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update", "position_data"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "diagnostics"]
```

### 5.3 Traffic Classification

```rust
/// Traffic classification for QoS routing
pub enum BANTrafficType {
    // Critical
    ExtremeVelocityDetection,
    FallDetection,
    EmergencyAlert,
    
    // High
    GaitAnomaly,
    StumblePrecursor,
    ThreatDetection,
    
    // Medium
    PresenceDetection,
    TrackingUpdate,
    PositionData,
    IMUOrientation,
    
    // Low
    BatteryStatus,
    HealthTelemetry,
    Diagnostics,
}

impl BANTrafficType {
    pub fn qos_class(&self) -> QoSClass {
        match self {
            Self::ExtremeVelocityDetection |
            Self::FallDetection |
            Self::EmergencyAlert => QoSClass::Critical,
            
            Self::GaitAnomaly |
            Self::StumblePrecursor |
            Self::ThreatDetection => QoSClass::High,
            
            Self::PresenceDetection |
            Self::TrackingUpdate |
            Self::PositionData |
            Self::IMUOrientation => QoSClass::Medium,
            
            Self::BatteryStatus |
            Self::HealthTelemetry |
            Self::Diagnostics => QoSClass::Low,
        }
    }
}
```

---

## 6. UWB Bandwidth Capabilities and Role Configuration

### 6.1 UWB Bandwidth Capabilities — Detailed

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
| Event camera burst | ~0.5-2 Mbps burst | ✅ Handled | Extreme velocity capture |
| Radar burst | ~1-5 Mbps burst | ✅ Handled | Raw radar data |
| Audio clip | ~0.1-0.5 Mbps | ✅ Handled | Compressed audio |

**Key insight:** UWB CAN handle 360° streaming at 2K-2.5K resolution, and CAN handle all 8 cameras simultaneously at QVGA-VGA. The architecture should not say "UWB can't do 360°" — it should say "UWB supports 360° at resolutions up to 2.5K sustained."

### 6.2 UWB Role Configuration

**Role: `timing_only`**

| Aspect | Specification |
|--------|---------------|
| Purpose | Sub-nanosecond time synchronization |
| Capability | Time sync between nodes, cm-level ranging |
| Data transfer | Minimal (timing packets only) |
| Power | ~50 mW |
| Use when | Precision timing needed, no high-bandwidth streaming |

**Role: `timing_and_ranging`**

| Aspect | Specification |
|--------|---------------|
| Purpose | Precision timing + distance measurement |
| Capability | All timing capabilities + ranging data |
| Data transfer | Occasional data bursts |
| Power | ~80 mW |
| Use when | Enhanced body-frame accuracy needed |

**Role: `full_bandwidth`**

| Aspect | Specification |
|--------|---------------|
| Purpose | Full streaming capability |
| Capability | All timing capabilities + sustained 4-6 Mbps streaming |
| Data transfer | Continuous high-bandwidth |
| Power | 100-150 mW sustained |
| Use when | Pendant streaming over UWB without WiFi |

**Configuration:**

```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"     # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false      # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5          # Conservative limit for thermal management
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

# Per-node UWB configuration
[nodes.pendant.uwb]
enabled = true
role = "full_bandwidth"
priority = "high"

[nodes.anklet_left.uwb]
enabled = true
role = "timing_and_ranging"

[nodes.anklet_right.uwb]
enabled = true
role = "timing_and_ranging"
```

### 6.3 Tiered Progressive Quality Strategy

**Principle:** Start functional, improve over time. Never blocked waiting for "perfect" data.

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

**Configuration:**

```toml
[pendant_360.progressive_quality]
enabled = true
baseline_resolution = "qvga"              # "qvga" | "vga" | "720p"
refinement_resolution = "720p"            # Target quality for progressive
refinement_interval_ms = 500              # How often to send refinement frames
adaptive_bandwidth = true                 # Adjust based on link quality
min_bandwidth_mbps = 1.0                  # Never go below this
max_bandwidth_mbps = 5.0                  # Never exceed this
```

---

## 7. Multi-Radio BLE vs Antenna Diversity

### 7.1 The Question

Should SENTINEL-WEAR use multiple BLE radios on the belt node for parallel connections to multiple nodes?

### 7.2 Multi-Radio BLE Analysis

**Theoretical benefits:**

| Benefit | Description |
|---------|-------------|
| Parallel connections | Each radio manages subset of nodes independently |
| Reduced per-radio scheduling | Simpler time management per radio |
| Higher aggregate bandwidth | Multiple simultaneous BLE links |
| Lower per-node latency | Dedicated radio per node or node group |

**Real-world problems:**

| Problem | Description |
|---------|-------------|
| RF self-interference | Radios centimeters apart on belt share spectrum |
| Receiver desensitization | Transmission on Radio A degrades Radio B's receiver |
| Antenna coupling | Antennas too close for isolation |
| Firmware complexity | Coordinated frequency hopping across radios |
| Power increase | Multiple radios = multiple power domains |
| PCB complexity | Multiple radio modules, multiple antenna designs |

**RF self-interference example:**

```
BLE Radio A transmits at 0 dBm
    ↓ (signal couples to nearby antennas)
BLE Radio B's receiver front-end partially saturates
    ↓ (sensitivity degrades by 10-20 dB)
BLE Radio B misses packets
    ↓ (retransmissions required)
Overall latency INCREASES instead of decreasing
```

### 7.3 When Multi-Radio BLE Makes Sense

Multi-radio BLE is appropriate ONLY when:

| Condition | Threshold |
|-----------|-----------|
| Node count | > 20-30 nodes |
| Physical spread | Room/building scale |
| Antenna separation | > 1 meter between radios |
| Body shadowing | Not the primary propagation challenge |

**Use cases:**
- Industrial sensor mesh (factory floor)
- Hospital patient monitoring (ward scale)
- Multi-user environment (conference center)
- Smart building (floor-wide deployment)

**NOT appropriate for:** Single-person wearable system with 4-6 nodes.

### 7.4 Antenna Diversity — The Correct Solution

**Principle:** Single BLE radio + Multiple antenna paths

```
Belt Node BLE Radio
    │
    ├── Antenna Switch
    │       ├── Left-side antenna (toward wearer's left)
    │       └── Right-side antenna (toward wearer's right)
    │
    └── Radio selects best antenna dynamically
```

**Key differences from multi-radio:**

| Aspect | Multi-Radio | Antenna Diversity |
|--------|-------------|-------------------|
| Transmitters | Multiple simultaneous | Single |
| Self-interference | Yes (problematic) | No |
| Power consumption | High (multiple radios) | Low (single radio + switch) |
| Complexity | High | Moderate |
| Reliability improvement | Marginal or negative | Significant |

### 7.5 Why Antenna Diversity Helps Wearables

**Body shadowing problem:**

The human body is a significant RF absorber at 2.4 GHz:
- Torso absorption: 10-30 dB attenuation
- Arm position changes: 5-15 dB variation
- Orientation changes: 10-20 dB variation

**Single antenna problem:**
```
Node on right side of body
    ↓ (body blocks signal)
Single antenna on left side of belt
    ↓ (poor signal)
High packet loss, retries, latency
```

**Dual antenna solution:**
```
Node on right side of body
    ↓
Left antenna: poor signal (-80 dBm)
Right antenna: good signal (-60 dBm)
    ↓
Radio switches to right antenna
    ↓
Reliable communication
```

### 7.6 Antenna Diversity Implementation

| Component | Specification |
|-----------|---------------|
| BLE radio | Single nRF5340 or equivalent |
| Antenna switch | SPDT RF switch (e.g., Skyworks SKY13330) |
| Switch time | < 1 microsecond |
| Antennas | 2× PCB trace or chip antennas, physically separated |
| Selection algorithm | RSSI-based or packet-loss-triggered |

**Configuration:**

```toml
[ban.antenna_diversity]
enabled = true
antenna_count = 2              # 1 = single, 2 = diversity
selection_mode = "rssi"        # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5        # Switch if other antenna is 5 dB better
preferred_antenna = "auto"     # "left" | "right" | "auto"
hysteresis_db = 2              # Don't switch too frequently
```

---

## 8. RF Coexistence

### 8.1 Frequency Bands in Use

| Radio | Frequency Band | Potential Conflicts |
|-------|---------------|---------------------|
| BLE | 2.4 GHz ISM (2400-2483.5 MHz) | WiFi 2.4 GHz, other BLE devices, microwaves |
| WiFi | 2.4 GHz (2400-2495 MHz) and 5 GHz (5150-5875 MHz) | BLE (2.4 GHz only) |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz (57-64 GHz) | None (separate band) |
| Cellular LTE | Various (700 MHz - 2.6 GHz) | Possible adjacent to 2.4 GHz |
| Cellular 5G Sub-6 | 3.5 GHz | Possible adjacent to UWB |
| Cellular 5G mmWave | 24-47 GHz | None (separate band) |

### 8.2 Coexistence Strategies

**Strategy 1: WiFi Prefers 5 GHz Band**

Most modern routers support 5 GHz. Configuring belt WiFi to prefer 5 GHz eliminates conflict with BLE.

```toml
[rf_coexistence]
wifi_prefer_5ghz = true        # Strongly recommended
```

**Strategy 2: Antenna Separation on Belt Node**

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

**Strategy 3: Time-Division Multiplexing (When 2.4 GHz WiFi Required)**

If 2.4 GHz WiFi must be used:
- BLE connection events scheduled during WiFi idle periods
- BLE gap during WiFi transmission
- Requires coordination between WiFi and BLE firmware

```toml
[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false     # Set true only if 2.4 GHz WiFi required
ble_gap_during_wifi_ms = 5     # Gap between BLE and WiFi if sharing
```

### 8.3 mmWave Radar (60 GHz) — No Conflict

mmWave radar at 60 GHz is in a completely separate frequency band from BLE, WiFi, UWB, and cellular. No coexistence concerns.

---

## 9. Data Contracts Between Nodes

### 9.1 What Is a Data Contract?

A data contract is the agreement between a node and the belt about:
- What data format the node will send
- How frequently
- With what latency guarantees
- What the belt will do with it

### 9.2 Anklet → Belt Contract

```rust
/// Data contract: Anklet node → Belt node
pub struct AnkletToBeltContract {
    /// Gait event data (every step)
    pub gait_event: Option<GaitEvent>,
    
    /// IMU sample rate (configurable, default 200 Hz)
    pub imu_sample_rate_hz: u16,
    
    /// IMU samples per packet (batched for efficiency)
    pub imu_samples_per_packet: u8,
    
    /// ToF reading (every 50 ms)
    pub tof_reading: Option<ToFReading>,
    
    /// Battery percentage (every 10 seconds)
    pub battery_percent: Option<u8>,
    
    /// Maximum latency requirement (for stumble detection)
    pub max_latency_ms: u16,
    
    /// QoS class for this node's traffic
    pub default_qos: QoSClass,
}

impl AnkletToBeltContract {
    pub fn default() -> Self {
        Self {
            gait_event: None,
            imu_sample_rate_hz: 200,
            imu_samples_per_packet: 20,  // 20 samples @ 200Hz = 100ms batches
            tof_reading: None,
            battery_percent: None,
            max_latency_ms: 50,
            default_qos: QoSClass::High,  // Gait is high priority
        }
    }
}
```

### 9.3 Pendant → Belt Contract

```rust
/// Data contract: Pendant node → Belt node
pub struct PendantToBeltContract {
    /// Presence/motion detection events
    pub detection_event: Option<DetectionEvent>,
    
    /// IMU orientation (every connection interval)
    pub imu_orientation: Option<Quaternion>,
    
    /// Acoustic event classification
    pub acoustic_event: Option<AcousticEvent>,
    
    /// Camera metadata (if camera active)
    pub camera_metadata: Option<CameraMetadata>,
    
    /// Raw media path (if stored locally)
    pub raw_media_path: Option<String>,
    
    /// Battery percentage (every 10 seconds)
    pub battery_percent: Option<u8>,
    
    /// Maximum latency requirement
    pub max_latency_ms: u16,
    
    /// Default QoS class
    pub default_qos: QoSClass,
}

impl PendantToBeltContract {
    pub fn default() -> Self {
        Self {
            detection_event: None,
            imu_orientation: None,
            acoustic_event: None,
            camera_metadata: None,
            raw_media_path: None,
            battery_percent: None,
            max_latency_ms: 30,
            default_qos: QoSClass::Medium,
        }
    }
}
```

### 9.4 Bracelet → Belt Contract

```rust
/// Data contract: Bracelet node → Belt node
pub struct BraceletToBeltContract {
    /// Motion detection event
    pub detection_event: Option<DetectionEvent>,
    
    /// IMU orientation (lower rate than pendant)
    pub imu_orientation: Option<Quaternion>,
    
    /// Camera metadata (if camera active and configured)
    pub camera_metadata: Option<CameraMetadata>,
    
    /// Battery percentage
    pub battery_percent: Option<u8>,
    
    /// Maximum latency
    pub max_latency_ms: u16,
    
    /// Default QoS class
    pub default_qos: QoSClass,
}

impl BraceletToBeltContract {
    pub fn default() -> Self {
        Self {
            detection_event: None,
            imu_orientation: None,
            camera_metadata: None,
            battery_percent: None,
            max_latency_ms: 50,
            default_qos: QoSClass::Medium,
        }
    }
}
```

---

## 10. Extreme Velocity Detection — Special Considerations

### 10.1 Latency Requirements

**Projectile flight times:**

| Projectile | Velocity | Flight Time (3 m) | Flight Time (10 m) | Flight Time (100 m) |
|------------|----------|-------------------|--------------------|---------------------|
| Rifle (850 m/s) | 850 m/s | 3.5 ms | 11.8 ms | 117.6 ms |
| Handgun (350 m/s) | 350 m/s | 8.5 ms | 28.6 ms | — |

**Detection latency budget:**

For realistic rifle engagement (100+ meters):
- Flight time: 117+ ms
- Detection + alert: 0.65-2.4 ms (with optimized system)
- Warning time: 115+ ms (abundant)

For close-range handgun (5-10 m):
- Flight time: 14-28 ms
- Detection + alert: 0.65-2.4 ms
- Warning time: 12-27 ms (useful but tight)

### 10.2 BLE Optimization for Extreme Velocity

**Critical QoS bypass:**

```rust
// When extreme velocity detection occurs:
// 1. Set QoS = Critical
// 2. Transmit immediately, preempting other traffic
// 3. Belt receives within 1-2 ms instead of up to 30 ms

// This reduces worst-case latency from ~35 ms to ~5 ms
```

**Reserved slot for critical alerts:**

```
BLE Connection Event (30 ms interval)
├── [Standard slots for normal traffic]
├── Slot 25-28 ms: Reserved (extreme velocity / critical alerts)
└── Slot 28-30 ms: Reserved (retransmits)
```

### 10.3 Haptic Latency — Actuator Choice

| Actuator Type | Onset Time | Suitable For |
|---------------|------------|--------------|
| ERM motor | 10-50 ms | ❌ Too slow for any scenario |
| LRA (linear resonant) | 2-10 ms | ✅ Rifle at 50+ m, handgun at 10+ m |
| Piezo haptic | 0.1-1.0 ms | ✅ All scenarios, including close-range |

**Recommendation:** LRA sufficient for realistic rifle distances. Piezo provides additional margin for close-range scenarios.

---

## 11. Power Budget per Transport

### 11.1 BLE Power

| State | Power | Notes |
|-------|-------|-------|
| Sleep | < 0.1 mW | Radio off |
| Idle (connected) | 1-5 mW | Waiting for connection event |
| RX active | 5-15 mW | Receiving |
| TX active | 10-20 mW | Transmitting |
| Average (typical use) | 3-10 mW | Depends on connection interval |

### 11.2 UWB Power

| State | Power | Notes |
|-------|-------|-------|
| Sleep | < 0.1 mW | Radio off |
| Idle (timing mode) | 10-30 mW | Timing sync only |
| Active (ranging) | 30-80 mW | Ranging + occasional data |
| Active (streaming) | 100-150 mW | Sustained high bandwidth |

### 11.3 WiFi Power (Belt Node Only)

| State | Power |
|-------|-------|
| Sleep | < 1 mW |
| Idle (connected) | 50-100 mW |
| Active (light traffic) | 150-300 mW |
| Active (streaming) | 300-500+ mW |

### 11.4 Cellular Power (Belt Node Only)

| State | Power |
|-------|-------|
| Sleep / Airplane mode | < 1 mW |
| Idle (registered) | 10-50 mW |
| RX/TX (light) | 150-400 mW |
| Active data | 400-800 mW (LTE Cat 4) |
| Active streaming | 600-1200 mW (5G) |

---

## 12. Configuration Reference

### 12.1 Complete BAN Configuration

```toml
# sentinel-wear.toml — BAN Configuration

[ban]
primary = "ble"                  # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30

[ban.ble_scheduling]
mode = "dynamic"                 # "static" | "dynamic"
default_connection_interval_ms = 30
anklet_interval_ms = 15
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100
sleep_interval_ms = 500

[ban.ble_scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true
extreme_velocity_preempt = true

[ban.uwb]
enabled = false
role = "timing_and_ranging"       # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[ban.uwb.offload]
camera_burst = true
radar_burst = true
audio_clip = true
slam_data = true
extreme_velocity_event_camera_burst = true

[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
reserved_slot_enabled = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor", "threat_detection"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update", "position_data"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "diagnostics"]

[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
preferred_antenna = "auto"
hysteresis_db = 2

[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false
ble_gap_during_wifi_ms = 5

# Node-specific configurations
[nodes.pendant.uwb]
enabled = true
role = "full_bandwidth"

[nodes.pendant_360.progressive_quality]
enabled = true
baseline_resolution = "qvga"
refinement_resolution = "720p"
refinement_interval_ms = 500
adaptive_bandwidth = true
min_bandwidth_mbps = 1.0
max_bandwidth_mbps = 5.0

[nodes.anklet_left.uwb]
enabled = true
role = "timing_and_ranging"

[nodes.anklet_right.uwb]
enabled = true
role = "timing_and_ranging"

# Extreme velocity specific
[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
direct_isr_to_radio = true
```

---

## 13. Summary

### 13.1 Key Principles

1. **BLE is the BAN backbone** — Lowest power, deterministic scheduling, sufficient for metadata and sensor data
2. **UWB for precision and bandwidth augmentation** — Timing, ranging, moderate streaming
3. **WiFi/Cellular for belt-to-app only** — Never on jewelry nodes
4. **Antenna diversity over multi-radio** — Single radio with dual antennas is the correct solution
5. **QoS classes for traffic prioritization** — Critical traffic preempts normal traffic
6. **Dynamic scheduling adapts to context** — Walking, threat, idle states adjust priorities

### 13.2 Latency Summary

| Scenario | BLE Latency | With UWB Offload | Notes |
|----------|-------------|------------------|-------|
| Standard detection | 10-30 ms | N/A | Sufficient for most use cases |
| High-priority detection | 5-20 ms | N/A | QoS High |
| Critical detection | 0.3-5 ms | N/A | QoS Critical + reserved slot |
| Camera streaming | N/A (WiFi) | 1-5 ms (UWB) | UWB at 2K-2.5K |
| 360° streaming | N/A (WiFi) | 1-5 ms (UWB) | UWB at 2K |

### 13.3 Power Summary

| Transport | Power (Active) | Best For |
|-----------|----------------|----------|
| BLE | 5-15 mW | Metadata, control, health |
| UWB | 50-150 mW | Timing, moderate streaming |
| WiFi | 300-500 mW | High-bandwidth streaming |
| Cellular | 150-1200 mW | Remote access |

---

**End of Document**
```
