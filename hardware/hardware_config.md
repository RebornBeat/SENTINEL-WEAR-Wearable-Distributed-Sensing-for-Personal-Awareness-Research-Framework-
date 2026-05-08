# Hardware Configuration Reference — SENTINEL-WEAR

**Version:** 0.3 | **Status:** Research Reference Design

## 1. Philosophy: No Artificial Limits

This document specifies the hardware interfaces, power architecture, connectivity hierarchy, and configuration standards for SENTINEL-WEAR sensing nodes. It is a **reference design**, not a constraint specification.

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at maximum speeds. Resolution, storage, compute are configuration choices.
- **Modularity:** Each functional block defined by interface, not part number.
- **Connectivity Agnostic:** BLE 5.x primary BAN transport; UWB optional for bandwidth and precision; Wi-Fi and Cellular for belt-to-app connectivity only.
- **Storage Agnostic:** microSD, internal flash, or network — user chooses.
- **User-Controlled Data Handling:** No system-imposed restrictions on storage, streaming, or retention.

---

## 2. Critical Architectural Constraint: Belt Node as Sole External Gateway

**Only the Belt Node has WiFi, Cellular, and direct BLE connection to the companion app.**

**All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via Body-Area Network (BAN) to the belt node.** They have no WiFi, no cellular, and no direct connection to external networks.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR Connectivity Model                         │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  ⚠️  BELT NODE IS THE ONLY NODE WITH WiFi / Cellular / BLE (to external)    │
│  ⚠️  ALL OTHER NODES = BAN ONLY (BLE/UWB to belt, nothing external)         │
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
| **Antenna** | Requires significant clearance | Small PCB antenna | Moderate clearance |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power/thermal/size requirements. BLE and UWB are the only viable BAN technologies for jewelry-scale wearables. All external connectivity (WiFi, Cellular) is concentrated at the belt node where larger form factor and battery are acceptable.

### Why NOT Multi-Radio BLE?

**Multi-radio BLE** (multiple BLE transceivers on the belt) sounds attractive for parallel connections, but causes more problems than it solves:

| Problem | Description |
|---------|-------------|
| **Self-interference** | Multiple transmitters in close proximity cause receiver desensitization |
| **RF coupling** | Antennas centimeters apart couple strongly, degrading sensitivity |
| **Increased power** | Multiple radios = multiple power draws |
| **Firmware complexity** | Coordinated frequency hopping, scheduling across radios |
| **Debugging difficulty** | Multi-radio RF problems are notoriously hard to diagnose |

**When multi-radio BLE IS appropriate:**
- Industrial sensor mesh (factory floor scale)
- Hospital patient monitoring (ward scale)
- Smart building (floor-wide deployment)
- Node count > 20-30

**NOT appropriate for SENTINEL-WEAR:** Single-person wearable with 4-6 nodes — the complexity exceeds benefits.

**The correct solution is antenna diversity** (see Section 7).

---

## 3. Transport Layer Hierarchy

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

UWB supports continuous streaming at appropriate resolutions. The following table clarifies what is achievable:

| Stream Type | Bandwidth Required | UWB Capability | Notes |
|-------------|-------------------|----------------|-------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled | Instant baseline for 360° |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled | Standard quality for all cameras |
| 8 cameras × 720p H.264 | ~8-15 Mbps | ❌ Exceeds sustained | Requires WiFi or burst mode |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled | Good quality panorama |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ✅ At max | High quality panorama |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Exceeds | Requires WiFi |
| Single camera 720p-1080p | ~1-3 Mbps | ✅ Handled | Forward camera stream |

**Tiered Progressive Quality Strategy:** When bandwidth is constrained, the system can start with low-resolution baseline and progressively improve:

```
Phase 1 (Instant):     8 cameras at QVGA → complete 360° view immediately
Phase 2 (Progressive): Send higher-res keyframes sequentially over 5-10 seconds
Phase 3 (Continuous):  Maintain low-res baseline, periodic high-res keyframes
Result:                Functional immediately, quality improves over time
```

**This is a capability, not a limitation.** The architecture supports graceful degradation and progressive improvement.

---

## 4. Core Node Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      SENTINEL-WEAR Node                           │
├──────────────────────────────────────────────────────────────────┤
│  MCU Core                                                         │
│  - ARM Cortex-M4F / M33 / M7 / Linux SoM (configurable)          │
│  - FPU required for IMU fusion                                    │
│  - DSP extensions preferred for radar and audio processing        │
├──────────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Node Type)                     │
│  - mmWave Radar (SPI/UART)                                        │
│  - IMU (I2C/SPI)                                                  │
│  - Acoustic / Microphone Array (I2S/PDM)                          │
│  - LiDAR/ToF (UART/SPI/I2C)                                       │
│  - Environmental (I2C)                                            │
│  - Event Sensor (MIPI/SPI parallel)                               │
│  - Camera (MIPI CSI-2 / DVP) — any variant, any node             │
│  - 360° Camera Array (multiple MIPI lanes + HW sync)              │
├──────────────────────────────────────────────────────────────────┤
│  Storage Interface (Optional Per Configuration)                   │
│  - microSD card (SPI, up to 2 TB) — raw recordings, SLAM maps    │
│  - Internal QSPI NOR Flash — metadata, logs, model weights        │
├──────────────────────────────────────────────────────────────────┤
│  Body-Area Network (BAN) Radio — ALL NON-BELT NODES              │
│  - BLE 5.x (mandatory, integrated or module)                      │
│  - UWB optional (Qorvo DW3000 class) for bandwidth and precision  │
│  - NO WiFi, NO Cellular                                           │
├──────────────────────────────────────────────────────────────────┤
│  Belt Node Additional — External Network Interfaces              │
│  - WiFi 802.11ac/ax (built into SoM or external module)           │
│  - Cellular LTE/5G module (nano-SIM or eSIM)                      │
│  - BLE 5.x (to companion app as fallback)                         │
│  - USB-C (PC companion + charging)                                │
├──────────────────────────────────────────────────────────────────┤
│  Power Management                                                 │
│  - Battery Charger (linear or switching)                          │
│  - Fuel Gauge (I2C)                                               │
│  - Buck/Boost Regulators                                          │
│  - Optional Belt Power Input (high-current from belt battery)     │
│  - Belt Node: Dual battery bays (hot-swap variant)                │
├──────────────────────────────────────────────────────────────────┤
│  User Interface (Optional Per Node)                               │
│  - Haptic Driver (I2C DRV2605L) + LRA or ERM actuator            │
│  - Status LEDs (GPIO)                                             │
│  - Hardware Power Switch (optional, user-installed) for cameras   │
├──────────────────────────────────────────────────────────────────┤
│  Debug Interface                                                  │
│  - SWD (Serial Wire Debug)                                        │
│  - UART Console (115200 baud)                                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. Body-Area Network (BAN) Architecture

### 5.1 BLE Scheduling and Determinism

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

### 5.2 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

**Configuration:**

```toml
[ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15              # Higher priority for gait detection
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100            # Optional node, lower priority
sleep_interval_ms = 500              # When all nodes in idle mode
```

### 5.3 Dynamic Scheduling — Context-Aware Adjustments

**Principle:** Schedule based on current activity context for optimal performance.

| Context | Schedule Adjustment | Rationale |
|---------|---------------------|-----------|
| Walking detected | Prioritize anklets | Gait timing is critical |
| Threat detected | Prioritize pendant | Sensing is critical |
| Idle state | Extend intervals | Power saving |
| Streaming active | Shift heavy traffic to UWB | Reduce BLE load |
| Extreme velocity alert | Preempt all other traffic | Critical latency requirement |

**Configuration:**

```toml
[ban.scheduling]
mode = "dynamic"                    # "static" | "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5  # Anklet slots increase by 50%
threat_pendant_priority_boost = 2.0  # Pendant slots double during threat
idle_interval_multiplier = 2.0       # Intervals double when idle
streaming_shift_to_uwb = true        # Heavy traffic to UWB
extreme_velocity_preempt = true      # Detection preempts all other slots
reserved_slot_start_ms = 25          # Reserved slot for critical alerts
reserved_slot_end_ms = 28
```

### 5.4 QoS Classes — Traffic Prioritization

**QoS Classification:**

| QoS Class | Priority | Traffic Types | Latency Guarantee |
|-----------|----------|---------------|-------------------|
| Critical | Highest | Extreme velocity detection, fall detection, emergency alerts | < 5 ms |
| High | Second | Gait anomaly, stumble precursor, threat detection | < 20 ms |
| Medium | Third | Presence detection, position tracking, classification | < 50 ms |
| Low | Fourth | Battery status, health telemetry, firmware update | < 500 ms |

**Critical QoS Behavior:**
- Preempts all other traffic immediately
- Can interrupt ongoing transmissions
- Guaranteed slot in every connection event
- Essential for extreme velocity detection latency requirements

**Configuration:**

```toml
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor", "threat_detection"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update", "classification"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "firmware_update"]
```

### 5.5 Node Priority for Bandwidth Allocation

When multiple nodes have data to transmit simultaneously, the belt node serves higher-priority nodes first:

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

**Override:** Critical QoS messages (fall detected, extreme velocity) are transmitted immediately regardless of priority slot.

### 5.6 UWB Role Configuration

UWB can be enabled on any node with UWB hardware (DW3000 class). Roles:

**Role: `timing_only`**
- Sub-nanosecond time synchronization between nodes
- cm-level ranging for body-frame reconstruction
- Minimal data transfer
- Lowest power UWB mode (~50 mW)
- Use when: precision timing needed, no high-bandwidth streaming

**Role: `timing_and_ranging`**
- All timing capabilities
- Distance measurement between nodes for body-frame accuracy
- Occasional data bursts
- Moderate power (~80 mW)
- Use when: enhanced body-frame accuracy needed

**Role: `full_bandwidth`**
- All timing capabilities
- Sustained streaming up to 5-6 Mbps
- Single camera streaming, 360° at 2K resolution
- Progressive quality strategies
- Higher power (100-150 mW sustained)
- Use when: pendant streaming over UWB without WiFi

**Configuration:**

```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"     # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false      # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5          # Conservative limit for thermal management
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]  # Which nodes have UWB populated
```

## 6. RF Coexistence

### 6.1 Frequency Bands in Use

| Radio | Frequency Band | Conflict Risk |
|-------|---------------|---------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz and 5 GHz | BLE (2.4 GHz only) |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz | None (separate band) |
| Cellular LTE | 700 MHz - 2.6 GHz | Possible adjacent to 2.4 GHz |
| Cellular 5G Sub-6 | 3.5 GHz | Possible adjacent to UWB |
| Cellular 5G mmWave | 24-47 GHz | None (separate band) |

### 6.2 Coexistence Strategies

**Strategy 1: WiFi Prefers 5 GHz Band**
- Most modern routers support 5 GHz
- BLE operates at 2.4 GHz — no conflict
- Strongly recommended configuration
- Eliminates the primary BLE/WiFi interference path

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
- Minimum recommended separation: 30-50 mm between any two 2.4 GHz antennas

**Strategy 3: Time-Division Multiplexing (When 2.4 GHz WiFi Required)**
- BLE connection events scheduled during WiFi idle periods
- BLE gap during WiFi transmission
- Requires coordination between WiFi and BLE firmware
- Only needed when 5 GHz WiFi unavailable

### 6.3 RF Isolation Requirements

For the belt node with multiple radios:

| Radio Pair | Isolation Target | Method |
|------------|------------------|--------|
| BLE ↔ WiFi (2.4 GHz) | > 20 dB | Antenna separation, frequency coordination |
| BLE ↔ Cellular (LTE Band 7) | > 15 dB | Antenna separation, filtering |
| WiFi ↔ Cellular | > 25 dB | Antenna separation, shielding if needed |
| UWB ↔ All | > 30 dB | Separate band, natural isolation |
| mmWave ↔ All | > 40 dB | Separate band (60 GHz), natural isolation |

### 6.4 Configuration

```toml
[rf_coexistence]
wifi_prefer_5ghz = true          # Strongly recommended
ble_wifi_timeshare = false       # Set true only if 2.4 GHz WiFi required
ble_gap_during_wifi_ms = 5       # Gap between BLE and WiFi if sharing
antenna_separation_mm = 40       # Minimum separation for belt node antennas
```

---

## 7. Antenna Diversity — Performance Enhancement

### 7.1 Problem: Body Shadowing

The human body is a significant RF absorber at 2.4 GHz:
- Torso absorption: 10-30 dB attenuation
- Arm position changes: 5-15 dB variation
- Orientation changes: 10-20 dB variation

**Single antenna problem:**
```
Node on right side of body
    ↓ (body blocks signal)
Single antenna on left side of belt
    ↓ (poor signal: -80 dBm)
High packet loss, retries, variable latency
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
Reliable communication (packet loss < 1%)
```

### 7.2 Solution: Antenna Diversity (NOT Multi-Radio)

**Key distinction:**

| Approach | Description | Self-Interference | Recommendation |
|----------|-------------|-------------------|-----------------|
| Multi-radio BLE | Multiple BLE transceivers | High — receiver desense | ❌ NOT recommended |
| Antenna diversity | ONE BLE radio + multiple antennas | None — only one transmitter | ✅ RECOMMENDED |

**Antenna diversity implementation:**
- Single BLE transceiver (nRF5340 or equivalent)
- SPDT or DPDT RF antenna switch (Skyworks SKY13330, Analog ADG918, or similar)
- Two antennas physically separated on the belt
- MCU GPIO controls antenna selection
- Firmware selects best antenna based on RSSI or packet loss
- Switch time: < 1 microsecond

### 7.3 Benefits of Antenna Diversity

| Benefit | Impact |
|---------|--------|
| Improved reliability | Reduces packet loss from body shadowing |
| Lower retries | More deterministic latency |
| Better coverage | Works regardless of body orientation |
| No self-interference | Only one transmitter active at a time |
| Critical for extreme velocity | Ensures detection messages arrive reliably |

### 7.4 Antenna Placement on Belt

```
Belt Node Interior Surface (Facing Body)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   [BLE Antenna LEFT]                         [BLE Antenna RIGHT]│
│   (Positioned toward wearer's left)           (toward right)     │
│        │                                           │              │
│        └──────────────► [RF Switch] ◄──────────────┘              │
│                           │                                      │
│                      [BLE Radio]                                  │
│                                                                 │
│   Physical separation: 60-80 mm recommended                      │
│   Both antennas on interior (body-facing) surface                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Placement guidelines:**
- Minimum separation: 40 mm (better: 60-80 mm)
- Both antennas on interior surface (body-facing)
- Avoid placement near metal battery
- Antenna types: PCB trace antenna or ceramic chip antenna
- Each antenna should have clear field of view to the body

### 7.5 Antenna Selection Algorithm

**RSSI-based selection (primary):**
```
1. Monitor RSSI on current antenna
2. If RSSI drops below threshold (-75 dBm):
   a. Measure RSSI on alternate antenna
   b. If alternate RSSI > current RSSI + hysteresis (3 dB):
      Switch to alternate antenna
   c. Else: Stay on current antenna
3. Repeat at next connection event
```

**Packet-loss-triggered selection (fallback):**
```
1. Track packet success rate on current antenna
2. If success rate < 90% over last 10 packets:
   a. Force switch to alternate antenna
   b. Reset counters
   c. Evaluate new antenna
```

### 7.6 Implementation Details

| Component | Specification | Notes |
|-----------|---------------|-------|
| BLE radio | nRF5340 or equivalent | Single transceiver |
| RF switch | SPDT, < 1 µs switch time | Skyworks SKY13330, Analog ADG918 |
| Control GPIO | 1-2 pins | From MCU to switch |
| Antennas | 2× PCB trace or chip | Physically separated |
| Switch algorithm | RSSI + packet loss | Dual trigger |

### 7.7 Why NOT Multi-Radio BLE

**Multi-radio BLE would mean:** Multiple BLE transceivers on the belt, each managing a subset of nodes.

**Problems with multi-radio BLE in wearable form factor:**

| Problem | Description | Severity |
|---------|-------------|----------|
| Self-interference | Multiple transmitters in close proximity cause receiver desensitization | High |
| RF coupling | Antennas centimeters apart couple strongly, degrading sensitivity 10-20 dB | High |
| Increased power | Multiple radios = 2-5× power consumption | Moderate |
| Firmware complexity | Coordinated frequency hopping, scheduling across radios | High |
| Debugging difficulty | Multi-radio RF problems are notoriously hard to diagnose | High |
| Increased cost | Multiple radio modules, crystals, matching networks | Moderate |

**When multi-radio BLE IS appropriate:**
- Industrial sensor mesh (factory floor scale)
- Hospital patient monitoring (ward scale)
- Smart building (floor-wide deployment)
- Node count > 20-30
- Physical antenna separation > 1 meter between radios

**NOT appropriate for SENTINEL-WEAR:** Single-person wearable with 4-6 nodes — the complexity exceeds benefits.

### 7.8 Per-Node Antenna Diversity (Optional)

Antenna diversity can also be implemented on other nodes:

**Bracelet:** Dual antenna (front/back of wrist) — helps with arm rotation effects

**Anklet:** Dual antenna (front/back of ankle) — helps with leg rotation during walking

**Configuration:**
```toml
[ban.antenna_diversity]
# Belt node antenna diversity (primary implementation)
enabled = true
antenna_count = 2
selection_mode = "rssi"            # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5            # Switch if other antenna is 5 dB better
hysteresis_db = 2                  # Prevent rapid switching
preferred_antenna = "auto"         # "left" | "right" | "auto"

# Per-node antenna diversity (if populated)
[nodes.bracelet_left.antenna_diversity]
enabled = false                    # Option for future variants

[nodes.anklet_left.antenna_diversity]
enabled = false                    # Option for future variants
```

---

## 8. Belt Node External Connectivity

### 8.1 WiFi Interface

| Specification | Value |
|---------------|-------|
| Standard | 802.11 a/b/g/n/ac/ax (depending on SoM) |
| Purpose | Primary companion app connection, media streaming, SLAM data transfer |
| Antenna | U.FL connector or integrated PCB antenna |
| Power | 300-500 mW when streaming |
| Bands | 2.4 GHz and 5 GHz (prefer 5 GHz for RF coexistence) |

**WiFi is present ONLY on the belt node.** No other node has WiFi capability.

**WiFi configuration:**
```toml
[connectivity.wifi]
enabled = true
prefer_5ghz = true              # Strongly recommended for RF coexistence
ssid = ""                       # Set at runtime or via config
password = ""                   # Set at runtime for security
```

### 8.2 Cellular / SIM Interface

The belt node PCB includes:
- nano-SIM card slot (4FF form factor) or eSIM footprint
- Cellular module interface (UART for Cat 1/4, USB 3.x for 5G)
- RF antenna connector (U.FL) for external cellular antenna
- Optional GNSS from cellular module

**Supported Cellular Modules:**

| Module | Technology | Speed | Interface | Power (Active) | Use Case |
|--------|------------|-------|-----------|----------------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | 150-300 mA | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | 400-600 mA | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | 800-1200 mA | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | 500-800 mA | Multi-carrier, eSIM |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Variable | Industrial grade |

**SIM Interface Signals:**

| Signal | Description | Voltage/Notes |
|--------|-------------|---------------|
| `SIM_CLK` | SIM card clock | Generated by cellular module |
| `SIM_DATA` | SIM card data | Bidirectional |
| `SIM_RST` | SIM card reset | Active low |
| `SIM_VCC` | SIM card power | 1.8V or 3.0V selectable |
| `SIM_DET` | SIM card present detect | GPIO input to MCU |

**Cellular is present ONLY on the belt node.** No other node has cellular capability.

**Cellular configuration:**
```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Stream video over cellular (data cost)
emergency_contact = ""           # Phone/endpoint for emergency alerts
```

### 8.3 Bluetooth Direct to Companion App

The belt node's BLE radio can connect directly to the companion app when:
- WiFi is unavailable
- User prefers BLE-only operation
- Minimal bandwidth sufficient (alerts, metadata)

**Limitations of BLE direct:**
- No video streaming (bandwidth insufficient)
- No SLAM data transfer
- Metadata and alerts only
- Useful for basic system interaction

**BLE direct configuration:**
```toml
[connectivity.bluetooth]
ble_direct_enabled = true        # Allow BLE connection to companion app
ble_advertising_name = "sentinel-wear"
```

### 8.4 USB-C Interface

| Function | Description |
|----------|-------------|
| PC companion app | Direct USB connection for desktop app |
| Firmware update | DFU mode for firmware flashing |
| Charging | USB-C PD preferred (5-20 V) |
| Debug | Serial console over USB CDC |

**USB-C is present ONLY on the belt node.**

### 8.5 Connectivity Summary

| Interface | Present On | Purpose |
|-----------|------------|---------|
| WiFi | Belt only | Primary app connection |
| Cellular | Belt only | Remote access, alerts |
| BLE (to app) | Belt only | Fallback app connection |
| USB-C | Belt only | PC connection, charging |
| BLE (BAN) | All nodes | Inter-node communication |
| UWB (BAN) | Optional per node | High-precision timing/bandwidth |

---

## 9. Test Pad Standard (12-Pad BAN Interface)

Standard test interface for all non-belt nodes:

| Pad | Signal | Description |
|---|---|---|
| 1 | `VCC_BATT` | Battery positive (3.7–4.2 V) |
| 2 | `GND` | Ground |
| 3 | `CHRG_IN+` | Charging input positive |
| 4 | `CHRG_IN-` | Charging input negative |
| 5 | `BAN_RF_TEST` | BAN radio test point (antenna tap) |
| 6 | `MCU_BOOT` | Boot mode select (HIGH = bootloader) |
| 7 | `MCU_RESET` | MCU reset (active low) |
| 8 | `CAM_HW_SW_TEST` | Camera hardware switch GPIO reading (if switch installed on this unit) |
| 9 | `HAPTIC_TEST` | Haptic actuator test point (AC coupled, LRA resonance) |
| 10 | `SWD_CLK` | SWD clock |
| 11 | `SWD_DATA` | SWD data |
| 12 | `UART_TX` | Debug UART TX (115200 baud) |

### Belt Node Additional Test Pads

| Pad | Signal | Description |
|---|---|---|
| 13 | `CELLULAR_PWR` | Cellular module power rail |
| 14 | `SIM_VCC` | SIM card power |
| 15 | `WIFI_RST` | WiFi module reset |
| 16 | `CELLULAR_STATUS` | Cellular module status LED / GPIO |
| 17 | `USB_D+` | USB data plus |
| 18 | `USB_D-` | USB data minus |
| 19 | `ANT_DIV_LEFT` | Left BLE antenna signal (for antenna diversity testing) |
| 20 | `ANT_DIV_RIGHT` | Right BLE antenna signal (for antenna diversity testing) |

### Test Pad Physical Layout

**Non-belt nodes:** 2×6 array (2 rows × 6 columns), 2.54 mm pitch

**Belt node:** 4×5 array (4 rows × 5 columns), 2.54 mm pitch, with clearly labeled test point IDs

---

## 10. 360° Curved Pendant Interface Standard

The 360° curved pendant requires additional interface definitions beyond the standard node header.

### 10.1 Camera Array Interface (Per Camera Module Position)

| Signal | Description | Routing Notes |
|---|---|---|
| `CAM_n_MIPI_CLK+` | MIPI CLK+ for camera n | 100 Ω differential, length-match within 2 mil |
| `CAM_n_MIPI_CLK-` | MIPI CLK- for camera n | Pair with above |
| `CAM_n_MIPI_D0+` | MIPI DATA0+ for camera n | 100 Ω differential |
| `CAM_n_MIPI_D0-` | MIPI DATA0- for camera n | |
| `CAM_FSYNC` | Hardware sync pulse broadcast to all cameras | Single GPIO from vision processor to all camera FSYNC pins, trace length matching required |
| `CAM_PWDN_n` | Power-down control per camera (optional) | GPIO per camera for selective shutdown |

**FSYNC Implementation:**
- Single FSYNC GPIO line from central vision processor
- Connects to FSYNC input of all cameras simultaneously
- All cameras trigger on same rising edge
- Target: < 10 ns skew between any two cameras
- Without FSYNC: cameras drift at frame rate, causing stitching artifacts

**FSYNC PCB requirements:**
- Length-matched traces to all camera FSYNC pins
- Maximum length variation: < 2 mm
- Series termination resistor (22-33 Ω) near vision processor

### 10.2 Vision Processor to Belt Node High-Speed Link

| Signal | Description | Speed |
|---|---|---|
| `USB3_TX+` | USB 3.x SuperSpeed TX differential + | 5 Gbps |
| `USB3_TX-` | USB 3.x SuperSpeed TX differential - | |
| `USB3_RX+` | USB 3.x SuperSpeed RX differential + | 5 Gbps |
| `USB3_RX-` | USB 3.x SuperSpeed RX differential - | |
| `USB3_GND` | USB 3.x ground reference | |

**Alternative high-speed links:**
- High-speed SPI (up to 50 MHz) for lower-bandwidth scenarios
- Proprietary SerDes link for maximum throughput
- MIPI CSI-2 (if belt node has MIPI receiver)

### 10.3 Belt Power Input (Extended Runtime)

| Signal | Description | Current Rating |
|---|---|---|
| `V_BELT_IN` | 5V input from belt node battery bank | Up to 2 A |
| `V_BELT_GND` | Return ground | |
| `V_BELT_EN` | Power enable signal (belt controls) | GPIO |

**Enables extended runtime operation where pendant draws power from belt's larger battery.**

**Implementation:**
- Conductive traces in necklace cable, or
- Separate thin power cable from belt to pendant
- Belt controller enables/disables power output
- Provides unlimited runtime for 360° pendant at full active mode

### 10.4 Stitching Architecture Options

**Option A: Pendant-side Stitching**
- Vision processor on pendant stitches all cameras to single 360° stream
- Single encoded stream transmitted to belt
- Reduces bandwidth requirement on link to belt
- Requires more compute on pendant
- Higher pendant power consumption
- Simpler belt-side processing

**Option B: Belt-side Stitching**
- All camera streams transmitted individually to belt
- Belt node stitches 360° panorama
- Higher bandwidth on link
- Pendant has simpler compute requirements
- Distributes compute load to belt

**Option C: Hybrid / Progressive**
- Low-res stitched baseline from pendant (immediate)
- Higher-res keyframes transmitted progressively
- Belt accumulates and improves quality over time
- Graceful degradation and progressive improvement

**Configuration:**
```toml
[nodes.pendant_360.stitching]
mode = "pendant_side"           # "pendant_side" | "belt_side" | "hybrid"
baseline_resolution = "vga"     # For hybrid mode baseline
keyframe_resolution = "720p"    # For hybrid mode keyframes
keyframe_interval_ms = 500     # How often to send high-res keyframes
```

### 10.5 Camera Coverage Configurations

| Config | Cameras | Angular Spacing | FoV Needed Per Camera | Total Bandwidth (VGA H.264) |
|--------|---------|-----------------|------------------------|---------------------------|
| Minimal | 4 | 90° | ≥ 100° | ~1-2 Mbps |
| Standard | 6 | 60° | ≥ 70° | ~1.5-3 Mbps |
| Dense | 8 | 45° | ≥ 55° | ~2-4 Mbps |
| Ultra | 12 | 30° | ≥ 40° | ~3-6 Mbps |

---

## 11. Power Architecture

### 11.1 Standard Nodes

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `VBAT` | 3.7–4.2V | LiPo battery | Direct to radio PA, haptic driver peak |
| `VCC_3V3` | 3.3V | LDO or buck from VBAT | MCU and all digital sensors |
| `VCC_1V8` | 1.8V | LDO | Low-power IMU, environmental |
| `V_CAM` | 3.3V or 2.8V | Regulated, optionally gated | Camera module(s) |

### 11.2 360° Pendant Additional Rails

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_CAMS` | 2.8V / 3.3V | Regulated, ganged | All N camera modules |
| `V_ISP` | 1.8V / 1.0V | Vision MCU supply | Vision processor core / IO |
| `V_EXT` | 5V | From belt node (optional) | Belt-supplied power for extended runtime |

### 11.3 Belt Node Additional Rails

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_CELLULAR` | 3.3V / 4.2V | Buck from main battery | Cellular module |
| `V_WIFI` | 3.3V | Built into SoM or external | WiFi module |

### 11.4 Power Domain Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           Power Architecture             │
                    └─────────────────────────────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
    ┌──────▼──────┐           ┌────────▼────────┐          ┌──────▼──────┐
    │   VBAT      │           │  USB-C / PoE   │          │ Belt Power  │
    │ 3.7-4.2V    │           │    5-20V       │          │   Output    │
    │ (Battery)   │           │  (Belt only)   │          │  (to pendant)│
    └──────┬──────┘           └────────┬────────┘          └──────┬──────┘
           │                           │                          │
           │              ┌────────────┼────────────┐             │
           │              │            │            │             │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──▼───┐ ┌──────▼──────┐      │
    │   3.3V      │ │   3.3V     │ │1.8V  │ │ 5V Output  │      │
    │   Buck     │ │   LDO      │ │ LDO  │ │  Enable   │      │
    │ (MCU/Sensors)│ │(Cellular) │ │(IMU) │ │  (GPIO)   │      │
    └──────┬──────┘ └────────────┘ └──────┘ └────────────┘      │
           │                                                   │
           │                                                   │
    ┌──────▼───────────────────────────────────────────────────▼──────┐
    │                    Node Components                              │
    │  MCU | Sensors | Radio | Haptic | Cameras (if present)         │
    └──────────────────────────────────────────────────────────────────┘
```

### 11.5 Node Weight and Battery Targets

| Node | Target Weight (g) | Battery (mAh) | Typical Runtime (Standard Mode) |
|---|---|---|---|
| Pendant — Standard | 30–60 | 150–300 | 24-48 hours |
| Pendant — 360° Curved | 50–120 | 400–1000 | 1-4 hours (full active) |
| Pendant — Medallion | 80–150 | 800–1500 | 12-24 hours |
| Bracelet — Minimal | 25–40 | 150–250 | 48-72 hours |
| Bracelet — Standard | 30–50 | 200–400 | 24-48 hours |
| Bracelet — Extended (Camera) | 40–60 | 300–500 | 12-24 hours |
| Belt — MCU (Variant A) | 100–150 | 2000 | 15-20 hours |
| Belt — Linux SoM (Variant B) | 150–250 | 5000-7000 | 10-15 hours (sparse), 5-8 hours (full) |
| Belt — Hot-swappable (Variant E) | 200–300 | 5000-7000 × 2 | Unlimited (with swaps) |
| Anklet — Minimal | 40–70 | 300 | 48-72 hours |
| Anklet — Extended | 50–100 | 400–500 | 24-48 hours |
| Eyewear — Clip-on | 5–15 | 50–80 | 8-16 hours |
| Eyewear — Frame/Headband | 10–30 | 80–150 | 6-12 hours |

*All values are research targets pending test measurement.*

### 11.6 Power Budget Analysis — Belt Node

| Mode | Functions Active | Power Draw | Runtime (5000 mAh) |
|------|------------------|------------|-------------------|
| Minimal (MCU-only belt) | BAN hub, sparse tracking | ~500 mW | ~37 hours |
| Standard (Linux, sparse) | BAN hub, sparse tracking, WiFi idle | ~1.4 W | ~13 hours |
| Full Active (Linux) | BAN hub, dense SLAM, 360° stitching, streaming | ~4.5 W | ~4 hours |
| Remote Streaming | Full Active + cellular | ~5.3 W | ~3.5 hours |

### 11.7 Power Budget Analysis — 360° Pendant

| Mode | Functions Active | Power Draw | Runtime (800 mAh) |
|------|------------------|------------|-------------------|
| Sparse sensing only | Radar + IMU, no cameras | ~500 mW | ~6 hours |
| Cameras active, standard quality | 8 cameras VGA, stitching | ~2.5 W | ~1.2 hours |
| Cameras active, high quality | 8 cameras 720p, stitching | ~4 W | ~0.7 hours |
| With belt power input | Full active | Unlimited (drawing from belt) | N/A |

### 11.8 Charging

| Charging Method | Nodes Supported | Power Source | Typical Charge Time |
|-----------------|-----------------|--------------|---------------------|
| USB-C | Belt, Pendant, Bracelet | USB-C charger (5V) | 1-3 hours |
| Qi Wireless | Anklet, Bracelet | Qi charging pad | 2-4 hours |
| Magnetic Pogo Pin | Pendant, Bracelet | Docking station | 1-2 hours |
| Belt Power Input | 360° Pendant (extended runtime) | Conductive chain/cable from belt | N/A (continuous) |

### 11.9 Hot-Swap (Belt Node Variant E)

Dual battery bays with:
- One battery always powering system
- Other battery can be swapped without interruption
- Requires dedicated power path management IC (e.g., TI BQ25895, Analog ADP5100)
- Charge-while-discharge capability
- Independent fuel gauges per battery

**Hot-swap configuration:**
```toml
[power.hot_swap]
enabled = false                   # Only for Variant E
low_battery_swap_threshold = 20  # Alert when battery < 20%
swap_time_s = 30                 # Time to swap battery
```

## 12. Thermal Management

### 12.1 Thermal Constraints for Wearables

**Maximum skin-contact temperature:** 40-42°C acceptable, higher causes discomfort.

**Key thermal generators:**
- Vision processor (360° pendant): 1-2 W
- MCU (all nodes): 0.1-0.5 W
- Radios (BLE/UWB): 0.05-0.15 W
- Cameras (360° pendant): 1-2 W (all 8 active)
- CW radar (extreme velocity detection): 0.2-0.5 W

### 12.2 Thermal Mitigation Strategies

**General (all nodes):**
- Thermal vias under heat-generating ICs
- Air gap between PCB and enclosure for convection
- Enclosure material selection (metal for heat spreading, plastic for insulation)

**360° Pendant specific:**
- Vision processor requires dedicated thermal pad or heat spreader
- Larger pendant enclosures have more surface area for dissipation
- Duty cycling cameras to reduce sustained thermal load
- Maximum pendant surface temperature target: 45°C (air side), 42°C (skin side)

**Belt Node specific:**
- Linux SoM generates 3-7 W under full load
- Thermal pad under SoM footprint to enclosure
- Vented enclosure design
- NTC thermistor for thermal monitoring
- Firmware throttles SLAM frame rate when enclosure exceeds 40°C

### 12.3 Thermal Monitoring

```toml
[thermal]
monitoring_enabled = true
thermistor_pin = "NTC1"
warning_threshold_c = 40
throttle_threshold_c = 45
shutdown_threshold_c = 50
throttle_action = "reduce_slam_rate"  # "reduce_slam_rate" | "reduce_camera_fps" | "disable_slam"
```

### 12.4 Thermal Response Under Extreme Velocity Detection

During extreme velocity detection:
- CW radar draws 0.2-0.5 W continuous (minimal thermal impact)
- Event camera draws 0.1-0.3 W when active (minimal thermal impact)
- Haptic burst (piezo): 0.5-1 W for < 100 ms (negligible sustained thermal impact)
- No special thermal mitigation required for detection events

---

## 13. Haptic Architecture — Directional and Multi-Modal Alerts

### 13.1 Haptic Actuator Types

| Actuator Type | Onset Time | Power | Suitability |
|---------------|------------|-------|--------------|
| ERM (rotating motor) | 10-50 ms | 50-200 mW | ❌ Too slow for extreme velocity |
| LRA (linear resonant) | 2-10 ms | 30-100 mW | ✅ Sufficient for most scenarios |
| Piezo haptic | 0.1-1.0 ms | 50-200 mW | ✅ Optimal for extreme velocity |
| Electrostatic skin stimulator | < 0.1 ms | 10-50 mW | ✅ Experimental, fastest |

### 13.2 Actuator Selection by Scenario

| Scenario | Distance | Flight Time | Recommended Actuator |
|----------|----------|-------------|----------------------|
| Rifle at 100+ m | 100-300 m | 117-353 ms | LRA sufficient (2-10 ms onset) |
| Rifle at 50-100 m | 50-100 m | 59-117 ms | LRA sufficient |
| Rifle at < 50 m | < 50 m | < 59 ms | Piezo preferred for maximum margin |
| Handgun at 10+ m | 10+ m | 28+ ms | LRA sufficient |
| Handgun at 5-10 m | 5-10 m | 14-28 ms | Piezo preferred |
| Handgun at < 5 m | < 5 m | < 14 ms | Piezo required for any warning |

### 13.3 Directional Haptic Routing

**Principle:** Alert the node closest to the threat direction.

| Threat Direction | Primary Haptic | Secondary Haptic |
|------------------|-----------------|------------------|
| Front | Pendant piezo | Both bracelets (light) |
| Front-left | Left bracelet (strong) | Pendant (light) |
| Left | Left bracelet (strong) | Left anklet (light) |
| Rear-left | Left anklet (strong) | Left bracelet (light) |
| Rear | Both anklets | Both bracelets (light) |
| Rear-right | Right anklet (strong) | Right bracelet (light) |
| Right | Right bracelet (strong) | Right anklet (light) |
| Front-right | Right bracelet (strong) | Pendant (light) |

### 13.4 Belt Node Haptic Configuration

All belt PCBs should include footprint for directional haptic array:

| Actuator Position | Purpose |
|-------------------|---------|
| Left side | Alert for left-hemisphere threats |
| Right side | Alert for right-hemisphere threats |
| Front | Alert for forward threats |
| Back | Alert for rear threats |

**Configuration:**

```toml
[haptic]
# Primary haptic type
type = "lra"                      # "lra" | "piezo" | "both"

# Piezo as enhancement (if populated)
piezo_enabled = false
piezo_onset_ms = 0.5
piezo_priority = "extreme_velocity_only"  # Use piezo only for extreme velocity

# LRA configuration
lra_onset_ms = 5
lra_pattern_library = true

# Directional haptics
[haptic.directional]
enabled = true
actuator_count = 4
alert_mode = "nearest_to_threat"
intensity_gradient = true         # Primary = strong, secondary = light
```

---

## 14. Camera Privacy Controls — Available Options

All camera privacy controls are user-selected. None are mandatory by the architecture.

### Option A: Hardware Power Switch (User-Installed)

- Physical switch cuts `V_CAM` or `V_CAMS` power rail
- When DISABLED: camera is physically unpowered. No software override.
- PCB provides footprint for this switch on all camera-capable nodes
- MOSFET on the `V_CAM` rail; switch controls gate drive
- Firmware reads `CAM_HW_SW_TEST` GPIO at boot if pin is populated
- Users who want unconditional physical camera disable select this option

**Implementation:**
```
Switch OFF → MOSFET gate LOW → V_CAM rail disconnected → Camera unpowered
Switch ON  → MOSFET gate HIGH → V_CAM rail connected → Camera powered
```

### Option B: Software-Only Control

- MCU controls `V_CAM` via MOSFET GPIO without any physical switch
- Software scheduling, detection-triggered activation, or manual control
- Simpler hardware (no switch component)
- Users who prefer flexibility over guaranteed physical disable select this option

### Option C: Always-On Camera

- Camera powered continuously; no gating
- Data handling (storage, streaming, retention) configured in `sentinel-wear.toml`
- Useful for continuous recording deployments

### Configuration

```toml
[privacy.camera]
hardware_switch_installed = false   # Set true if physical switch is present
software_control_enabled = true
default_state = "disabled"          # "enabled" | "disabled"
schedule = []                        # List of time ranges when enabled, empty = always
trigger_on_detection = false         # Enable camera on detection event
```

---

## 15. Sensing Interfaces

### 15.1 mmWave Radar

| Signal | Direction | Description |
|---|---|---|
| `RADAR_nRST` | Output | Hardware reset |
| `RADAR_nCS` | Output | SPI chip select |
| `RADAR_IRQ` | Input | Data ready interrupt |
| `SPI_MOSI/MISO/CLK` | Bidirectional | Shared SPI bus |

**Operating modes:**

| Mode | Use Case | Latency |
|------|----------|---------|
| FMCW (standard) | Presence detection, tracking | 50-200 ms update rate |
| FMCW (extended Doppler) | Extreme velocity detection | 10-20 ms update rate |
| CW (continuous wave) | Primary trigger for extreme velocity | 10-100 kHz detection rate |

**Extended Doppler configuration for projectile detection:**

Standard automotive profiles support ±18 m/s. For projectile detection (±300 m/s):
- Custom chirp configuration required
- Reduced range resolution acceptable
- TI IWR6843 supports this via custom chirp parameters

### 15.2 CW Radar Interface (Extreme Velocity Detection)

Dedicated interface for continuous-wave radar mode:

| Signal | Direction | Description |
|--------|-----------|-------------|
| `CW_TX_EN` | Output | Enable CW transmission |
| `CW_IF_OUT` | Input | Intermediate frequency output |
| `CW_IF_GAIN` | Output | Programmable gain control |

**CW mode advantages for extreme velocity:**
- No chirp latency
- Pure Doppler detection
- 50-150 µs detection time
- Ideal for Tier 1 reflex trigger

### 15.3 IMU

- I²C (≤ 1 MHz Fast Mode Plus) or SPI (≤ 10 MHz)
- `IMU_DRDY` interrupt for new-sample wake
- High dynamic range required for anklets (5-10 g for heel-strike)

### 15.4 Acoustic (Microphone Array)

- I²S or PDM (MCU provides MCLK and BCLK)
- 3–6 microphones supported (topology per node variant)
- On-node processing recommended to reduce bandwidth

### 15.5 LiDAR / ToF

- UART (common for integrated modules) or SPI
- `LIDAR_TRIG` for external synchronization with other sensors

### 15.6 Environmental

- I²C, shared bus with IMU
- Temperature, humidity, pressure, VOC, PM2.5

### 15.7 Event Sensor

- MIPI CSI-2 or proprietary SPI-like parallel
- DMA required for high-bandwidth event rates
- Used for extreme velocity detection (microsecond latency)
- **Corrected latency:** 10-100 µs for event generation and transmission (not 1 ms)

### 15.8 Camera

- MIPI CSI-2 (standard single camera or 360° array)
- 100 Ω differential impedance; length-match within 2 mil per pair
- DVP (8-bit parallel) for lower-resolution lower-bandwidth modules
- FSYNC for multi-camera synchronization

---

## 16. Storage Architecture

### 16.1 Local Storage Options

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD card | Up to 2 TB | SPI | Raw video, SLAM maps, long recordings |
| QSPI NOR Flash | 8-128 MB | QSPI | Metadata, logs, model weights |
| Internal MCU Flash | 512 KB - 2 MB | Internal | Firmware, critical config |

### 16.2 Belt Node Storage

| Storage Type | Capacity | Interface | Use Case |
|--------------|----------|-----------|----------|
| microSD card | Up to 2 TB | SDIO (high-speed) | Aggregated recordings, SLAM maps |
| NVMe SSD | 128 GB - 2 TB | PCIe (x86 variants) | Production deployment |
| USB storage | Variable | USB 3.x | Backup, archive |

### 16.3 Extreme Velocity Evidence Storage

When extreme velocity is detected:

| Data Type | Storage Location | Retention |
|-----------|------------------|-----------|
| Detection timestamp + velocity + direction | Node flash + belt SD | Permanent |
| Event camera recording | Pendant SD + belt SD | Per retention policy |
| Conventional camera clip (if present) | Belt SD | Per retention policy |
| Full characterization log | Belt SD | Permanent with integrity chain |

### 16.4 Configuration

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
include_integrity_chain = true   # SHA-256 hash chain for legal evidence

[data.extreme_velocity]
store_event_camera_burst = true
store_trajectory_data = true
integrity_chain = true
```

---

## 17. Battery Chemistry & Safety

### 17.1 Requirements

All nodes use LiPo cells at 3.7V nominal:
- UL 1642 or IEC 62133 certified cells required
- Overcharge protection: MOSFET cutoff at 4.25V
- Overdischarge protection: cutoff at 3.0V
- Thermal protection: NTC thermistor, cutoff above 60°C during charge
- Short-circuit protection: PTC fuse or dedicated protection IC

### 17.2 Charging

| Charging Method | Nodes Supported | Power Source |
|-----------------|-----------------|--------------|
| USB-C | Belt, Pendant, Bracelet | USB-C charger (5V) |
| Qi Wireless | Anklet, Bracelet | Qi charging pad |
| Magnetic Pogo Pin | Pendant, Bracelet | Docking station |
| Belt Power Input | 360° Pendant (extended runtime) | Conductive chain or cable from belt |

### 17.3 Hot-Swap (Belt Node Variant E)

Dual battery bays with:
- One battery always powering system
- Other battery can be swapped without interruption
- Requires dedicated power path management IC

---

## 18. Hypoallergenic Material Requirements

EN 1811:2011 nickel release compliance for all skin-contact surfaces:

**Compliant Materials:**
- Grade 5 titanium (Ti-6Al-4V) — preferred metal
- 316L surgical stainless steel — alternative metal
- Medical-grade silicone (ISO 10993) — band material
- Hard-coat anodized aluminum — enclosures (coating must not chip)
- Polycarbonate or PETG — plastic components

**Non-Compliant (Do Not Use for Skin-Contact):**
- 6061/7075 aluminum alloys
- Brass
- Unplated zinc die-cast
- Standard (non-surgical) stainless steel

---

## 19. Debug Interface

### 19.1 Standard Debug

10-pin ARM Cortex Debug Connector (0.05" pitch) or Tag-Connect footprint:
- `SWDIO`, `SWCLK` — Serial Wire Debug
- `nRST` — MCU Reset
- `SWO` — Serial Wire Output (printf over SWO)
- `UART_TX`, `UART_RX` — Console at 115200 baud

### 19.2 Bootloader

- USB DFU for firmware updates (nodes with USB)
- ROM bootloader for OTA updates (nodes without USB)
- Boot mode selection via `MCU_BOOT` test pad

### 19.3 Production Debug Disable

After production calibration:
- Debug interface can be permanently disabled via fuse/OTP
- Prevents unauthorized firmware extraction
- User choice: research units may leave debug enabled

---

## 20. Extreme Velocity Detection Hardware

### 20.1 Realistic Engagement Distance Analysis

**Key insight:** Detection range is more important than detection latency for realistic scenarios.

| Category | Distance | Weapons | Flight Time | Warning Time Needed |
|----------|----------|---------|-------------|---------------------|
| Point-blank | < 5 m | All | < 15 ms | Physically constrained |
| Close-range | 5-15 m | Handguns | 15-45 ms | 10-30 ms useful |
| Medium-range | 15-50 m | All | 45-200 ms | Comfortable margins |
| Long-range | 50-300 m | Rifles | 60-350 ms | Abundant margins |

### 20.2 Detection Range vs Latency Tradeoff

| Configuration | Target Range | Range Resolution | Latency | Use Case |
|---------------|--------------|------------------|---------|----------|
| Range-optimized | 150 m | 2 m | 5 ms | Rifle detection at distance |
| Latency-optimized | 50 m | 0.3 m | 1 ms | Close-range maximum margin |
| Balanced | 100 m | 0.5 m | 2 ms | General deployment |

### 20.3 Tiered Detection Architecture

**Tier 1 — Reflex Trigger (50-150 µs):**

| Parameter | Value |
|-----------|-------|
| Sensor | CW radar (primary) |
| Processing | Doppler FFT threshold |
| Output | Binary detection, local haptic trigger |
| Purpose | Immediate awareness |

**Tier 2 — Direction Validation (100-500 µs):**

| Parameter | Value |
|-----------|-------|
| Sensor | Event camera (primary) |
| Processing | Streak analysis, angle estimation |
| Output | Velocity vector (magnitude + direction) |
| Purpose | Directional alert routing |

**Tier 3 — Characterization (5-50 ms):**

| Parameter | Value |
|-----------|-------|
| Sensor | FMCW burst + conventional cameras |
| Processing | Range-Doppler + trajectory |
| Output | Full threat characterization + evidence |
| Purpose | Forensic documentation |

### 20.4 Latency Budget — Realistic Scenarios

**Rifle at 100 m:**

| Stage | Latency | Cumulative |
|-------|---------|------------|
| Flight time | 117.6 ms | — |
| Detection (CW radar) | 50-150 µs | ~0.1 ms |
| BLE transmission (standard) | 0-30 ms | 0.1-30 ms |
| Haptic onset (LRA) | 2-10 ms | 2.1-40 ms |
| **Minimum warning** | **77 ms** | |
| **Maximum warning** | **115 ms** | |

**Conclusion:** Even with worst-case standard BLE + LRA haptics, warning time at 100 m is 77-115 ms — well within human reaction time.

### 20.5 CW Radar Configuration

```toml
[extreme_velocity.detection]
# Primary trigger configuration
mode = "cw"                       # "cw" | "fmcw"

[extreme_velocity.detection.cw]
fft_size = 64
fft_window_us = 32
adc_sample_rate_hz = 2000000
velocity_resolution_ms = 77
velocity_threshold_ms = 50
snr_threshold_db = 10

[extreme_velocity.detection.range]
# Range optimization priority
optimization = "balanced"         # "range" | "latency" | "balanced"
target_range_m = 100
range_resolution_m = 0.5
```

### 20.6 Event Camera Integration

**Corrected latency understanding:** Event cameras have 10-100 µs latency for event generation and transmission, not 1 ms.

| Parameter | Value |
|-----------|-------|
| Event generation latency | 10-50 µs |
| Event transmission | 10-50 µs |
| Streak detection | 50-200 µs |
| Angle estimation | 50-200 µs |
| **Total** | **100-500 µs** |

**Event camera on pendant:**
- Positioned at forward angles for threat direction detection
- Asynchronous event emission
- DMA transfer to MCU memory
- Streak analysis for trajectory estimation

### 20.7 Firmware Configuration

```toml
[extreme_velocity]
enabled = false                 # Opt-in feature
mode = "production"             # "disabled" | "research" | "production"
doppler_update_rate_hz = 10000  # High update rate for fast transients
event_camera_always_on = true
min_velocity_ms = 50            # Ignore objects below this velocity
max_detection_distance_m = 100  # Detection range target
alert_on_detection = true
processing_priority = "realtime" # Preempts other processing

[extreme_velocity.alert]
# Alert configuration
haptic_type = "auto"            # "lra" | "piezo" | "auto" (select based on distance)
directional_routing = true
nodes = ["pendant", "bracelet_left", "bracelet_right"]

[extreme_velocity.recording]
trigger_conventional_cameras = true
capture_duration_s = 2.0
include_integrity_chain = true
```

---

## 21. Interface Summary Tables

### 21.1 Node Connectivity Summary

| Node Type | BLE | UWB | WiFi | Cellular | Notes |
|-----------|-----|-----|------|----------|-------|
| Pendant | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| 360° Pendant | ✅ Mandatory | ⚠️ Recommended | ❌ None | ❌ None | UWB for bandwidth |
| Bracelet | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Anklet | ✅ Mandatory | ⚠️ Recommended | ❌ None | ❌ None | UWB for gait sync |
| Eyewear | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Belt | ✅ Mandatory | ⚠️ Optional | ✅ Mandatory | ⚠️ Optional | Sole external gateway |

### 21.2 Bandwidth Capability by Transport

| Transport | Max Sustained | Max Burst | Primary Use |
|-----------|---------------|-----------|-------------|
| BLE 5.x | ~500 Kbps | ~2 Mbps | Control, metadata |
| UWB | ~5 Mbps | ~6.8 Mbps | Timing, moderate streaming |
| WiFi | 50-300+ Mbps | N/A | High bandwidth |
| LTE Cat 1 | 5-10 Mbps | N/A | Alerts, metadata |
| LTE Cat 4 | 50-150 Mbps | N/A | Moderate streaming |
| 5G | 100-1000+ Mbps | N/A | Live 360° streaming |

### 21.3 Extreme Velocity Detection Latency Summary

| Scenario | Distance | Flight Time | Detection + Alert | Warning Time |
|----------|----------|-------------|-------------------|---------------|
| Rifle (850 m/s) | 3 m | 3.5 ms | 0.65-2.4 ms (piezo) | 1.1-2.85 ms |
| Rifle (850 m/s) | 50 m | 58.8 ms | 2.5-11.5 ms (LRA) | 47-56 ms |
| Rifle (850 m/s) | 100 m | 117.6 ms | 2.5-11.5 ms (LRA) | 106-115 ms |
| Rifle (850 m/s) | 300 m | 352.9 ms | 2.5-11.5 ms (LRA) | 341-350 ms |
| Handgun (350 m/s) | 10 m | 28.6 ms | 2.5-11.5 ms (LRA) | 17-26 ms |

---

## 22. Configuration File Reference

Complete configuration example:

```toml
# sentinel-wear.toml

[nodes]
pendant = { sensors = ["MmWaveRadar", "Acoustic", "Imu"], variant = "standard" }
pendant_360 = { sensors = ["MmWaveRadar", "Acoustic", "Imu", "Camera360"], variant = "curved", enabled = false }
bracelet_left = { sensors = ["MmWaveRadar", "Imu", "Haptic"], variant = "standard" }
bracelet_right = { sensors = ["MmWaveRadar", "Imu", "Haptic"], variant = "standard" }
belt = { sensors = ["MmWaveRadar", "Imu", "EnvSensor"], variant = "linux_som" }
anklet_left = { sensors = ["LidarTof", "Imu", "Haptic"], variant = "standard" }
anklet_right = { sensors = ["LidarTof", "Imu", "Haptic"], variant = "standard" }
eyewear = { sensors = ["EventSensor", "Imu"], variant = "clip_on", enabled = false }

[connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""

[connectivity.bluetooth]
ble_direct_enabled = true

[ban]
primary = "ble"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

[ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100
sleep_interval_ms = 500

[ban.scheduling]
mode = "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true
extreme_velocity_preempt = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28

[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor", "threat_detection"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update", "classification"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "firmware_update"]

[ban.uwb]
enabled = false
role = "timing_and_ranging"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
hysteresis_db = 2
preferred_antenna = "auto"

[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false
ble_gap_during_wifi_ms = 5

[haptic]
type = "lra"
piezo_enabled = false
lra_onset_ms = 5

[haptic.directional]
enabled = true
actuator_count = 4
alert_mode = "nearest_to_threat"
intensity_gradient = true

[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "sd_card"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0
include_integrity_chain = true

[data.extreme_velocity]
store_event_camera_burst = true
store_trajectory_data = true
integrity_chain = true

[privacy.camera]
hardware_switch_installed = false
software_control_enabled = true
default_state = "disabled"
schedule = []
trigger_on_detection = false

[thermal]
monitoring_enabled = true
warning_threshold_c = 40
throttle_threshold_c = 45
shutdown_threshold_c = 50
throttle_action = "reduce_slam_rate"

[extreme_velocity]
enabled = false
mode = "disabled"
doppler_update_rate_hz = 10000
event_camera_always_on = false
min_velocity_ms = 50
max_detection_distance_m = 100
alert_on_detection = true
processing_priority = "realtime"

[extreme_velocity.detection]
mode = "cw"
optimization = "balanced"
target_range_m = 100

[extreme_velocity.alert]
haptic_type = "auto"
directional_routing = true
nodes = ["pendant", "bracelet_left", "bracelet_right"]

[extreme_velocity.recording]
trigger_conventional_cameras = true
capture_duration_s = 2.0
include_integrity_chain = true

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""
```
