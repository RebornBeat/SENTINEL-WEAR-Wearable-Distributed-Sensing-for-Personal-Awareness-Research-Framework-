# Belt Node — Primary Compute, Hub, and Sole External Gateway

**Project:** SENTINEL-WEAR
**Node Position:** Waist belt / belt clip / belt pack
**Primary Role:** Compute hub, battery hub, torso IMU reference, BAN coordinator, **sole external network gateway**, app server, recording manager, SLAM processor
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Role in the Mesh — The Sole External Gateway

### 1.1 Architectural Constraint (Not Configuration)

The Belt Node is the **central coordination unit** of the SENTINEL-WEAR system and the **only node with external network connectivity**. This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via Body-Area Network (BAN) to the belt node.** They have no WiFi, no cellular, and no direct connection to the companion app.

### 1.2 Why This Architecture?

| Constraint | WiFi on Wearable Node | BLE on Wearable Node | UWB on Wearable Node |
|------------|----------------------|---------------------|---------------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Form factor impact** | Requires larger battery, antenna space, thermal management | Minimal | Moderate |
| **Wearability** | Breaks jewelry form factor | Preserves jewelry form | Compatible |
| **Determinism** | Variable latency, contention, RF chaos | Consistent, schedulable | Excellent timing |
| **Antenna** | Requires significant clearance | Minimal antenna needs | Moderate antenna needs |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power/thermal/size requirements. The belt node, worn at the waist with a larger enclosure, is the only node capable of housing WiFi and cellular hardware without compromising the jewelry aesthetic of other nodes.

### 1.3 Data Flow Architecture

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

### 1.4 Primary Functions

| Function | Description | Power Impact |
|----------|-------------|--------------|
| **Compute Hub** | Runs `sentinel-fusion`, `sentinel-tracking` (PentaTrack bridge), `sentinel-slam` (Linux SoM variant), and all system coordination | 200-500 mW (sparse) to 2000-3000 mW (SLAM) |
| **Torso Reference** | The belt IMU defines the body-frame origin (+Y forward, +X right, +Z up). All other node orientations are expressed relative to this | 5-10 mW (IMU only) |
| **BAN Hub** | Routes all inter-node BAN traffic. Receives all detections, gait events, acoustic events, identification results, and recording notifications from all nodes | 50-150 mW (BLE), 100-300 mW (BLE+UWB) |
| **External Network Gateway** | The **sole** point of external connectivity. WiFi (home network primary) and optional cellular SIM (remote access, emergency contact). No other node connects to any external network | 300-500 mW (WiFi active), 150-1200 mW (cellular) |
| **App Server** | Runs the embedded HTTP/WebSocket server for the SENTINEL-WEAR companion app. Serves REST API, WebSocket event stream, and media streams | 100-500 mW (depends on load) |
| **Recording Manager** | Aggregates recording availability notifications from all nodes. Manages belt-local recording storage (SD card). Serves recordings to companion app. Generates legal export packages | 50-200 mW (during I/O) |
| **Sensing** | mmWave radar (downward, torso-level) for independent ground-plane and proximity coverage | 100-300 mW |
| **SLAM Processor** | (Linux SoM variants) Runs 360° panorama stitching from curved pendant cameras; runs SLAM pipeline for dense world model | 500-2000 mW |

---

## 2. Connectivity Architecture Detail

### 2.1 WiFi (Primary External Path)

- **Standard:** 802.11 a/b/g/n/ac/ax (depending on SoM)
- **Use:** Local network connectivity for companion app; high-bandwidth media streaming; SLAM data transfer
- **Bandwidth:** 50-300+ Mbps (variant-dependent)
- **Power:** 300-500 mW when actively streaming, 50-100 mW idle connected
- **Not present on:** Any other node in the mesh

**WiFi configuration:**
```toml
[connectivity.wifi]
enabled = true
prefer_5ghz = true              # Strongly recommended for RF coexistence with BLE
ssid = ""                        # Set at runtime for security
password = ""                    # Set at runtime for security
fallback_to_ble = true          # Use BLE direct if WiFi unavailable
```

### 2.2 Cellular / SIM (Secondary External Path — Optional)

- **Purpose:** Remote access when away from home WiFi; emergency contact alert delivery; cellular-primary deployments (rural, mobile)
- **SIM types supported:** nano-SIM (physical) or eSIM (embedded)

#### Supported Cellular Modules

| Module | Technology | Speed | Interface | Power (Active) | Use Case |
|--------|------------|-------|-----------|----------------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | 150-300 mA | Alerts, metadata, minimal data |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | 300-500 mA | Compressed video clips, moderate streaming |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | 500-1000 mA | Live 360° streaming, high bandwidth |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | 400-700 mA | Multi-carrier, eSIM, no physical SIM swap |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | 500-1200 mA | Industrial grade, rugged deployment |

#### Data Plan Guidance

| Use Case | Typical Monthly Data | Recommended Module |
|----------|---------------------|-------------------|
| Alerts and metadata only | < 100 MB | EC21 (LTE Cat 1) |
| Compressed video clips (occasional) | 1-5 GB | EC25 (LTE Cat 4) |
| Moderate live streaming | 10-30 GB | EC25 or RM502Q |
| 360° live streaming (frequent) | 30-100+ GB | RM502Q (5G) |

#### Cellular Configuration

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular when active
stream_via_cellular = false      # Stream video/audio via cellular (data cost consideration)
emergency_contact = ""           # Phone number or endpoint for emergency alerts
emergency_contact_trigger = "manual"   # "manual" | "fall_detected" | "critical_alert"
```

#### Cellular Physical Hardware

The belt node PCB includes:
- nano-SIM card slot (standard 4FF form factor) OR eSIM footprint (depending on variant)
- Cellular module connected via UART (Cat 1/4) or USB 3.x (5G modules)
- RF antenna connector (U.FL) for external cellular antenna routed to enclosure edge
- GNSS capability (if cellular module includes GNSS) for emergency contact location
- Power management for cellular module (can be independently powered down)

### 2.3 Bluetooth (Direct Local — Fallback)

- **Standard:** BLE 5.3
- **Use:** Direct companion app connection when WiFi unavailable
- **Capabilities:** Alerts, configuration, pairing, firmware updates, metadata
- **Limitations:** **Not** sufficient for video streaming, 360° panorama, SLAM data
- **Power:** 20-50 mW when connected

**BLE direct connection is a fallback for minimal operation, not a primary path for rich media.**

### 2.4 Transport Layer Priority

The belt node selects transport in this priority order:

| Priority | Transport | Condition | Use |
|----------|-----------|-----------|-----|
| 1 | WiFi (local) | Connected to home network | Full capability, all streaming |
| 2 | Cellular | WiFi unavailable, cellular enabled | Alerts, compressed video, moderate streaming |
| 3 | BLE direct | No WiFi, no cellular | Alerts, configuration, metadata only |

---

## 3. Body-Area Network (BAN) Hub Architecture

### 3.1 BAN Protocol Roles

The belt node is the **BAN coordinator** for all other nodes. It:
- Establishes and maintains BLE connections to all nodes
- Manages connection intervals and scheduling
- Serves as time master for network synchronization
- Routes data between nodes and to external networks

### 3.2 BLE Connection Scheduling

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
├── Slot 25-28 ms: Reserved (critical alerts, extreme velocity)
└── Slot 28-30 ms: Reserved (retransmits, eyewear)
```

**Note:** Slot 25-28 ms is reserved for critical alerts including extreme velocity detection. This guarantees a transmission opportunity every cycle for time-critical messages.

**Connection interval tradeoffs:**

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

**Configuration:**
```toml
[ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15              # Higher priority for gait
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100            # Optional node, lower priority
sleep_interval_ms = 500              # When all nodes in idle mode
reserved_slot_start_ms = 25          # Reserved slot for critical alerts
reserved_slot_end_ms = 28
```

### 3.3 Dynamic Scheduling

**Context-aware scheduling** adjusts priorities based on current activity:

| Context | Schedule Adjustment | Implementation |
|---------|---------------------|----------------|
| Walking detected | Increase anklet priority | `anklet_interval_ms = 15` |
| Threat detected | Increase pendant priority | `pendant_interval_ms = 15` |
| Idle state | Reduce all intervals | `default_connection_interval_ms = 100` |
| Streaming active | Shift heavy traffic to UWB | `uwb_offload = true` |
| Extreme velocity detection | Immediate transmission (QoS Critical) | Preempt other traffic |

**Configuration:**
```toml
[ban.scheduling]
mode = "dynamic"                   # "static" | "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true
```

### 3.4 QoS Classes for BLE Traffic

**Traffic prioritization** ensures critical messages are delivered with minimal latency:

| QoS Class | Priority | Traffic Types | Behavior |
|-----------|----------|---------------|----------|
| **Critical** | Highest | Extreme velocity detection, fall detection, emergency alerts | Preempts all other traffic, immediate transmission |
| **High** | Second | Gait timing, stumble precursor, threat detection | Sent in next available slot, preferential queue |
| **Medium** | Third | Presence detection, position tracking, identifications | Normal scheduled slot |
| **Low** | Fourth | Battery status, health telemetry, diagnostics | Sent only when bandwidth available |

**Implementation:**

```rust
pub enum BANQoSClass {
    Critical,   // Preempts all other traffic, immediate transmission
    High,       // Sent in next available slot, preferential queue
    Medium,     // Normal scheduled slot
    Low,        // Sent only when bandwidth available
}

// Critical messages (extreme velocity detection) bypass normal scheduling
// They can interrupt ongoing transmissions (with coordination)
```

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
traffic_types = ["presence_detection", "tracking_update", "identification"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "diagnostics"]
```

### 3.5 Node Priority for Bandwidth Allocation

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

**How priority affects behavior:**
- When multiple nodes have data to transmit, belt serves highest priority first
- Lower priority nodes may experience slight latency during high-traffic periods
- Critical QoS messages override priority (any node can immediately transmit critical alert)

### 3.6 Time Synchronization

The belt node is the **time master** for the entire BAN:
- Broadcasts synchronized timestamps to all nodes at each BLE connection event
- Each node runs `omni-sense-time::ClockOffsetEstimator` for drift tracking
- UWB (if enabled) provides sub-nanosecond synchronization for SLAM accuracy

### 3.7 UWB Integration (Optional)

The belt node can coordinate UWB across nodes that have UWB capability:

**UWB roles:**
- **Timing only:** Sub-nanosecond time synchronization across all nodes
- **Timing + ranging:** cm-level distance measurement between belt and other nodes
- **Timing + bandwidth:** Above plus moderate-bandwidth data transfer (single camera, compressed clips)

**UWB configuration:**
```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"       # "timing_only" | "timing_and_ranging" | "full_bandwidth"
nodes_with_uwb = ["pendant"]      # Which nodes have UWB capability
streaming_fallback = false         # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5             # Conservative limit for thermal management
```

---

## 4. Antenna Diversity Architecture

### 4.1 Why Antenna Diversity

**The problem:** Human body shadowing at 2.4 GHz causes significant signal attenuation:
- Torso absorption: 10-30 dB attenuation
- Arm position changes: 5-15 dB variation
- Orientation changes: 10-20 dB variation

**Single antenna limitation:** A node on one side of the body may have poor connectivity to a single belt antenna on the other side.

**Solution:** Single BLE radio with multiple antenna paths (antenna diversity), NOT multiple BLE radios.

### 4.2 Antenna Diversity vs Multi-Radio BLE

| Approach | Description | Self-Interference | Power | Complexity |
|----------|-------------|-------------------|-------|------------|
| **Antenna Diversity** | Single BLE radio, multiple antennas, dynamic selection | None | Low | Moderate |
| **Multi-Radio BLE** | Multiple BLE transceivers, each with own antenna | High risk | High | Very High |

**Why multi-radio BLE is NOT recommended:**
- Multiple BLE radios share the same 2.4 GHz spectrum
- Antennas are centimeters apart on the belt
- Self-interference causes receiver desensitization
- Net result: Worse performance, not better

**Antenna diversity is the correct architectural choice for wearable BLE reliability.**

### 4.3 Antenna Diversity Implementation

```
Belt Node BLE Architecture

    BLE Radio (Single)
         │
         ├── Antenna Switch (SPDT)
         │       ├── Left Antenna (PCB trace or chip, left side of belt)
         │       └── Right Antenna (PCB trace or chip, right side of belt)
         │
         └── Selection Algorithm
                 ├── RSSI comparison
                 ├── Packet loss tracking
                 └── Hysteresis (prevent rapid switching)
```

**Antenna selection logic:**
1. Monitor RSSI from both antennas
2. If alternate antenna RSSI > current antenna RSSI + threshold (5 dB)
3. Switch to alternate antenna
4. Apply hysteresis: Do not switch back unless difference exceeds threshold + hysteresis (7 dB total)

### 4.4 Configuration

```toml
[ban.antenna_diversity]
enabled = true                     # Enable dual-antenna diversity
antenna_count = 2                  # 1 = single, 2 = diversity
selection_mode = "rssi"            # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5            # Switch if other antenna is 5 dB better
hysteresis_db = 2                  # Don't switch back unless 7 dB better
preferred_antenna = "auto"         # "left" | "right" | "auto"
scan_interval_ms = 100             # How often to compare antenna RSSI
```

### 4.5 Impact on Extreme Velocity Detection

Antenna diversity improves reliability of critical alert delivery:
- Reduces packet loss from body shadowing
- Fewer retries → deterministic latency
- Reliable transmission regardless of body orientation
- Critical when detection occurs during movement

---

## 5. Compute Variants

### Variant A — MCU-Class (Minimal)

**Compute Platform:**
- **MCU:** STM32H743 or equivalent (Cortex-M7, 480 MHz, 2 MB Flash, 1 MB RAM)
- Sufficient for: full body-frame fusion (6 nodes), PentaTrack, alert routing, BAN hub, basic recording management

**Connectivity:**
- BLE: nRF52840 co-processor or integrated BLE module
- WiFi: External ESP32-S3 co-processor (WiFi used only as transport to companion app)
- Cellular: Quectel EC21 via UART (optional)

**Storage:**
- microSD card slot (SPI mode) for recordings
- External flash for firmware and configuration

**Power:**
- Battery: 2000-3000 mAh
- Runtime: 12-20 hours (sparse mode)
- Limitations: No SLAM, no 360° stitching, limited simultaneous streams

**Use case:** Basic presence awareness, low-cost deployment, minimal feature set

### Variant B — Linux SoM (Standard / Full Capability)

**Compute Platform:**
- **SoM:** Raspberry Pi Compute Module 4 (quad-core Cortex-A72, 2.4 GHz, 4-8 GB RAM, built-in WiFi/BT)
- Runs full Linux, all crates including `sentinel-slam`, `sentinel-api` (axum HTTP server)
- Full companion app server: REST API, WebSocket, RTSP, H.264, 360° streams

**Connectivity:**
- BLE: nRF5340 co-processor for real-time BAN coordination
- WiFi: Built-in 802.11ac (CM4) or external module
- Cellular: Quectel EC25 or RM502Q via USB

**Capabilities:**
- 360° panorama stitching from curved pendant cameras
- SLAM-ready: ORB-SLAM3, LIO-SAM, or similar
- Multiple simultaneous camera streams
- Neural inference (via external accelerator if needed)

**Power:**
- Battery: 5000-7000 mAh
- Runtime: 10-15 hours (sparse mode), 5-8 hours (full active with SLAM and streaming)
- Requires thermal management

**Use case:** Standard deployment, full capability, most common configuration

### Variant C — High-Performance SoM

**Compute Platform:**
- **SoM:** NXP i.MX 8M Plus (quad-core Cortex-A53 + Cortex-M7, NPU 2.3 TOPS)
- All Variant B capabilities plus:
  - NPU enables on-device neural classification without external accelerator
  - M7 co-processor handles real-time BAN without interrupting A53 fusion
  - Hardware video encoding acceleration

**Connectivity:**
- BLE: Integrated or co-processor
- WiFi: 802.11ac/ax
- Cellular: 5G module (RM502Q-AE) for live 360° streaming via cellular

**Power:**
- Battery: 5000-7000 mAh
- Runtime: 5-8 hours (full active with SLAM, 360° streaming, 5G)
- Thermal management critical

**Use case:** Production deployment, maximum capability, professional users

### Variant D — ESP32-Based (Minimal Cost)

**Compute Platform:**
- **MCU:** ESP32-S3 (Xtensa LX7 dual-core, 240 MHz, integrated WiFi + BLE)
- Lowest cost option, sufficient for 2-3 nodes
- WiFi native to MCU (used only as BAN/app transport)

**Connectivity:**
- WiFi: Integrated 802.11 b/g/n
- BLE: Integrated BLE 5.0
- Cellular: Quectel EC21 via UART (optional)

**Power:**
- Battery: 2000 mAh
- Runtime: 10-15 hours (sparse mode)

**Limitations:** No SLAM, limited simultaneous node support, no 360° stitching

**Use case:** Minimal deployment, research, cost-constrained applications

### Variant E — Extended Runtime / Hot-Swappable

**Compute Platform:**
- Same as Variant B (Linux SoM) or Variant C (High-Performance)
- Dual battery bays with hot-swap capability

**Battery Architecture:**
- Two independent battery bays
- Each bay: 3000-4000 mAh Li-Ion
- User can swap one battery while running on the other
- Total capacity: 6000-8000 mAh

**Form Factor:**
- Larger belt pack style (not buckle replacement)
- Dimensions: ~120 mm × 80 mm × 30 mm
- Weight: 200-400 g including batteries

**Power:**
- Runtime: Unlimited with battery swaps (each swap cycle adds 10-20 hours)
- Continuous operation for security shifts, professional use

**Use case:** Professional deployment, 24/7 operation, security personnel, shift workers

---

## 6. MCU Co-Processor Pattern (Variants B, C, E)

For Linux SoM variants, a dedicated co-processor handles real-time BAN tasks that cannot tolerate Linux scheduler latency:

### 6.1 Co-Processor Role

| Task | Why Co-Processor | Latency Requirement |
|------|------------------|---------------------|
| BLE connection management | Real-time scheduling, connection events | Milliseconds |
| Time synchronization | Precise timestamp broadcast | Microseconds |
| BAN packet routing | Deterministic handling | Milliseconds |
| Sensor I/O (belt IMU, radar) | Real-time data collection | Tens of milliseconds |
| Antenna diversity switching | Real-time RSSI monitoring | Milliseconds |

### 6.2 Co-Processor Options

**For Variant B (CM4):**
- nRF5340 (BLE 5.3, network core handles BLE, application core handles coordination)
- Interface: SPI or UART to CM4

**For Variant C (i.MX 8M Plus):**
- Integrated Cortex-M7 serves as co-processor
- Internal bus connection to A53 cores

### 6.3 Firmware Split

**Co-processor firmware (`no_std` Rust):**
- BLE stack and connection scheduling
- Time synchronization master
- BAN packet routing with QoS handling
- Antenna diversity management
- Sensor data collection

**Linux SoM software:**
- All `sentinel-*` crates
- SLAM, API server, recording manager
- High-level coordination and user interaction

---

## 7. Hardware Interfaces Summary

### 7.1 External Connectivity (Belt Node Only)

| Interface | Purpose | Present on Belt Only? |
|---|---|---|
| WiFi (802.11) | Companion app, media streaming | **Yes — belt only** |
| Cellular SIM | Remote access, emergency contact | **Yes — belt only** |
| BLE 5.3 (to external) | Direct app connection (fallback) | **Yes — belt only** |
| USB-C | PC companion app, firmware update, charging | Yes |

### 7.2 BAN Connectivity

| Interface | Purpose | Present on All Nodes? |
|---|---|---|
| BLE 5.3 (BAN) | All inter-node communication | Yes (BAN transport) |
| UWB (optional) | Precision time-sync + ranging + bandwidth | Optional on all nodes |

### 7.3 Sensing Interfaces

| Interface | Purpose | Belt-Specific? |
|---|---|---|
| mmWave Radar (downward) | Ground-level presence, trip hazards | Yes |
| IMU (torso reference) | Body-frame origin | Yes |
| Environmental | Temperature, humidity, pressure, VOC | Yes |

### 7.4 Storage Interfaces

| Interface | Purpose | Belt-Specific? |
|---|---|---|
| microSD card | Primary recording storage | Yes (primary aggregation point) |
| Internal flash | Firmware, configuration | Yes |

### 7.5 RF Interfaces

| Interface | Purpose | Notes |
|---|---|---|
| WiFi antenna | U.FL connector for external antenna | Exterior of enclosure |
| Cellular antenna | U.FL connector for external antenna | Exterior of enclosure |
| BLE antenna(s) | PCB trace or chip antenna | Interior (facing body), dual for diversity |
| UWB antenna (optional) | PCB trace or chip antenna | Interior |

---

## 8. Downward-Facing mmWave Radar

### 8.1 Purpose

- Ground-level object detection (trip hazards, approaching feet)
- Surface detection for gait analysis cross-reference
- Independent proximity sensing at torso level
- Support for extended Doppler configuration (extreme velocity detection)

### 8.2 Module Options

| Module | Frequency | Features | Power | Notes |
|--------|-----------|----------|-------|-------|
| TI IWR6843AOP | 60 GHz | Antenna-on-package, FMCW, good angular resolution, **configurable Doppler range** | 200-400 mW | Preferred for production, supports extended Doppler for projectile detection |
| Acconeer XR112 | 60 GHz | Ultra-compact, SPI, low power | 30-100 mW | Preferred for space-constrained |

### 8.3 Extended Doppler Configuration

For extreme velocity detection (projectile detection up to 850 m/s):

**Standard automotive profile:** ±18 m/s Doppler range (too slow for bullets)

**Extended Doppler configuration:**
- Trade range resolution for velocity range
- TI IWR6843 can be configured for ±300 m/s with reduced range
- Requires custom chirp parameters in firmware

**Configuration:**
```toml
[sensors.radar]
model = "IWR6843"
mode = "standard"               # "standard" | "extended_doppler"

[sensors.radar.extended_doppler]
enabled = false                  # Enable for extreme velocity detection
max_velocity_ms = 300           # Maximum detectable velocity in m/s
range_resolution_m = 0.5        # Reduced range resolution (tradeoff)
```

### 8.4 Coverage Geometry

- Mounted on bottom face of belt enclosure
- Downward angle: 15-30° from horizontal
- Field of view: ~120° horizontal, ~30° vertical
- Range: 0.2-5 m effective for ground-level detection

---

## 9. IMU (Torso Reference — Critical)

### 9.1 Why This IMU Is Critical

The belt IMU defines the **body-frame origin** for the entire system:
- All other node orientations are expressed relative to this
- Body-frame fusion accuracy depends on belt IMU noise characteristics
- A 10% improvement in belt IMU noise propagates to improved tracking across all nodes

### 9.2 Recommended Sensors

| Sensor | Noise Density | Power | Notes |
|--------|---------------|-------|-------|
| Bosch BMI270 | 0.1 mg/√Hz | 0.2 mW | Good balance of performance and power |
| TDK ICM-42688-P | 0.07 mg/√Hz | 0.3 mW | Best-in-class noise, smallest package |
| ST LSM6DSV | 0.1 mg/√Hz | 0.15 mW | Built-in step/gesture detection |

### 9.3 Configuration

```toml
[sensors.imu]
model = "ICM-42688-P"
sample_rate_hz = 200
full_scale_g = 16
low_noise_mode = true
```

---

## 10. Power Architecture

### 10.1 Power Sources

| Source | Variant | Capacity | Charging |
|--------|---------|----------|----------|
| Single Li-Ion 18650 | A, D | 2000-3500 mAh | USB-C |
| Dual Li-Ion 18650 (parallel) | B, C | 5000-7000 mAh | USB-C PD |
| Dual hot-swap bays | E | 6000-8000 mAh each | USB-C PD, hot-swap capable |

### 10.2 Power Domains

| Domain | Voltage | Usage | Control |
|--------|---------|-------|---------|
| `V_MAIN` | 5.0V | Raw input from USB/battery | Unswitched |
| `V_MCU` | 3.3V | MCU, digital logic | Always on |
| `V_RADIO_BLE` | 3.3V | BLE radio | Always on (BAN essential) |
| `V_RADIO_WIFI` | 3.3V | WiFi module | Software controlled |
| `V_RADIO_CELLULAR` | 3.8-4.2V | Cellular module | Software controlled |
| `V_SENSOR` | 3.3V | Radar, IMU, environmental | Software controlled (can sleep) |
| `V_STORAGE` | 3.3V | SD card | Software controlled |

### 10.3 Power Budget by Operating Mode

**Variant B (Linux SoM) Power Budget:**

| Mode | Functions Active | Power Draw | Runtime (5000 mAh) |
|------|------------------|------------|-------------------|
| **Minimal (MCU-only equivalent)** | BAN hub, sparse tracking, no WiFi/SLAM | ~500 mW | ~37 hours |
| **Standard Sparse** | BAN hub, sparse tracking, WiFi idle, BLE active | ~800 mW | ~23 hours |
| **Standard Connected** | BAN hub, sparse tracking, WiFi connected, API server idle | ~1.2 W | ~15 hours |
| **Standard Streaming** | Above + single camera stream | ~1.8 W | ~10 hours |
| **Full Active (SLAM + 360°)** | BAN hub, dense SLAM, 360° stitching, WiFi streaming | ~4.5 W | ~4 hours |
| **Full Active + Cellular** | Above + 5G cellular streaming | ~5.3 W | ~3.5 hours |

**Variant A (MCU-only) Power Budget:**

| Mode | Power Draw | Runtime (2000 mAh) |
|------|------------|-------------------|
| Minimal (sparse only) | ~300 mW | ~25 hours |
| Standard | ~500 mW | ~15 hours |
| With cellular (alert-only) | ~700 mW | ~10 hours |

**Variant E (Hot-Swappable) Power Budget:**

| Mode | Power Draw | Time Between Swaps (4000 mAh per bay) |
|------|------------|---------------------------------------|
| Standard Sparse | ~1 W | ~15 hours |
| Full Active | ~5 W | ~3 hours |

### 10.4 Power Management Strategies

**Battery-aware feature scaling:**

```toml
[power.battery_aware]
enabled = true
# Reduce features as battery depletes
scale_slam_on_low_battery = true      # Reduce SLAM frame rate below 50%
scale_streaming_on_low_battery = true # Reduce streaming quality below 30%
disable_slam_on_critical = true       # Disable SLAM below 15%
disable_streaming_on_critical = true  # Disable streaming below 10%
```

**Thermal-aware power management:**

```toml
[power.thermal]
max_enclosure_temp_c = 45            # Maximum comfortable temperature
throttle_slam_above_temp_c = 40      # Reduce SLAM above this temperature
disable_slam_above_temp_c = 45       # Disable SLAM to prevent overheating
monitor_ntc = true                    # Use NTC thermistor
```

---

## 11. Thermal Management

### 11.1 Heat Sources

| Source | Heat Generation | When Active |
|--------|----------------|-------------|
| SoM (Linux) | 2-5 W | Full compute |
| WiFi module | 0.3-1 W | Streaming |
| Cellular module | 0.5-2 W | Active connection |
| SLAM processing | 1-3 W | Dense mapping |
| 360° stitching | 1-2 W | Pendant active |
| **Total (full active)** | **5-10 W** | Maximum load |

### 11.2 Thermal Design Requirements

**For Linux SoM variants (B, C, E):**

1. **Thermal pad** under SoM footprint (connect to copper pour)
2. **Copper pour** on PCB for heat spreading
3. **Vented enclosure** or thermal spreader (aluminum plate) on enclosure exterior
4. **NTC thermistor** mounted near SoM for temperature monitoring
5. **Firmware throttling** reduces SLAM frame rate above 40°C

**Enclosure temperature limits:**
- Interior (component side): Up to 60°C acceptable
- Exterior (skin contact): Maximum 42°C for comfort
- Thermal design must ensure skin-contact temperature stays below limit

### 11.3 Thermal Mitigation Strategies

| Strategy | Implementation | Tradeoff |
|----------|----------------|----------|
| Duty cycling | SLAM runs 50% of the time during high temp | Reduced SLAM update rate |
| Quality scaling | Reduce SLAM resolution during high temp | Reduced map detail |
| Streaming quality reduction | Lower video bitrate during high temp | Reduced video quality |
| Feature disable | Disable SLAM above threshold | No dense mapping |
| Active cooling (Variant E only) | Small fan in belt pack | Noise, power draw |

---

## 12. RF Coexistence

### 12.1 Frequency Bands in Use

| Radio | Frequency | Potential Conflicts |
|-------|-----------|---------------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz and 5 GHz | BLE (at 2.4 GHz) |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz | None (separate band) |
| Cellular | 700 MHz - 3.5 GHz, 24-47 GHz (5G mmWave) | Minor interaction possible |

### 12.2 Coexistence Strategies

**Strategy 1: WiFi prefers 5 GHz band**
- Eliminates conflict with BLE (which uses 2.4 GHz)
- Default configuration

**Strategy 2: Antenna separation on belt enclosure**
- Cellular/WiFi antennas on exterior of belt (facing outward)
- BLE/UWB antennas on interior (facing body)
- Physical separation reduces coupling

**Strategy 3: Antenna diversity for BLE**
- Dual BLE antennas (left and right side of belt)
- Single BLE radio selects best antenna dynamically
- Eliminates body shadowing without multi-radio complexity

**Strategy 4: Time-division multiplexing**
- If 2.4 GHz WiFi must be used, coordinate timing with BLE connection events
- BLE gaps during WiFi transmission

```toml
[rf_coexistence]
wifi_prefer_5ghz = true        # Strongly recommended
ble_wifi_timeshare = false     # Set true if 2.4 GHz WiFi must be used
ble_gap_during_wifi_ms = 5     # Gap between BLE and WiFi if sharing 2.4 GHz
```

### 12.3 Antenna Placement

```
Belt Node Enclosure (Cross-Section)

    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   [Cellular Antenna]              [WiFi Antenna]            │  ← Exterior
    │   (U.FL to external)              (U.FL to external)        │
    │                                                             │
    │   ┌───────────────────────────────────────────────────┐   │
    │   │              PCB / Electronics                     │   │
    │   │                                                   │   │
    │   │   [BLE Antenna L]            [BLE Antenna R]      │   │  ← Interior
    │   │   (PCB trace or chip)         (PCB trace or chip) │   │     (toward body)
    │   │                                                   │   │
    │   └───────────────────────────────────────────────────┘   │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                          Wearer's body
```

---

## 13. Storage Architecture

### 13.1 Storage Options

| Storage | Capacity | Use |
|---------|----------|-----|
| microSD card (removable) | Up to 2 TB | Primary recording storage, removable for physical access |
| Internal eMMC (Linux SoM) | 8-32 GB | Firmware, configuration, temporary storage |
| External USB drive | Unlimited | Extended storage for large deployments |

### 13.2 Recording Management

The belt node aggregates recordings from all nodes:
- Receives `RecordingAvailable` BAN events from nodes
- Indexes all recordings in local database
- Serves recordings via API to companion app
- Enforces retention policy
- Generates legal export packages with integrity chain

### 13.3 Storage Configuration

```toml
[storage]
primary = "sd_card"                # "sd_card" | "internal" | "usb"
retention_days = 30                # 0 = keep forever
max_storage_mb = 0                 # 0 = unlimited
auto_export = false                # Auto-export to network storage
integrity_chain = true             # Include cryptographic chain in all recordings
```

---

## 14. No Identification Layer

The Belt Node does **not** carry an identification sensor:
- No camera
- No biometric sensor
- No hardware switch for identification purposes

**Reason:** The belt node is the compute and network hub. Identification is the role of the Pendant Node (user-configured). Separating identification from the central hub:
- Maintains a clear separation of concerns
- Prevents the most critical node from having privacy-sensitive components
- Allows users to choose identification capability independently of compute capability

---

## 15. Form Factor Options

### 15.1 Belt Buckle Replacement

- **Dimensions:** 80 mm × 60 mm × 20 mm
- **Mounting:** Replaces standard belt buckle
- **Weight:** 100-150 g (without extended battery)
- **Aesthetic:** Brushed aluminum or titanium, professional appearance
- **Limitations:** Limited internal volume, thermal constraints
- **Variants:** A, D (MCU-class)

### 15.2 Rear Belt Pack

- **Dimensions:** 100 mm × 70 mm × 25 mm
- **Mounting:** Clips to belt at rear hip
- **Weight:** 150-300 g including battery
- **Advantages:** More internal volume, better thermal dissipation, larger battery
- **Variants:** B, C (Linux SoM)

### 15.3 Extended Belt Pack (Variant E)

- **Dimensions:** 120 mm × 80 mm × 30 mm
- **Mounting:** Belt pack style
- **Weight:** 200-400 g including dual batteries
- **Advantages:** Hot-swap batteries, maximum runtime, active cooling option
- **Variants:** E only

### 15.4 Charging

| Variant | Charging Method | Charge Time |
|---------|-----------------|-------------|
| A, D | USB-C (5V/2A) | 2-3 hours |
| B, C | USB-C PD (9V/2A) | 2-3 hours |
| E | USB-C PD (9V/3A) | 1-2 hours per battery |

### 15.5 Antenna Placement

- Cellular/WiFi antennas on enclosure exterior (U.FL connectors to external antennas)
- BLE/UWB antennas on PCB interior (facing body)
- Dual BLE antennas for diversity (left and right sides)
- Antenna connectors accessible for replacement/upgrade

---

## 16. Extreme Velocity Detection — Latency Optimization

### 16.1 Latency Requirements

| Projectile Type | Engagement Distance | Time to Impact | Detection Budget |
|-----------------|---------------------|----------------|------------------|
| Rifle (850 m/s) | 3 m | 3.5 ms | < 1 ms ideal |
| Handgun (350 m/s) | 3 m | 8.5 ms | < 2 ms ideal |

### 16.2 Latency Breakdown

| Stage | Latency | Notes |
|-------|---------|-------|
| Doppler detection | 10-100 µs | Hardware event |
| Event camera trigger | < 1 ms | Fast transient detection |
| MCU processing | 100-500 µs | Detection classification |
| BLE queue (without QoS) | 0-30 ms | Depends on schedule slot |
| BLE queue (with QoS Critical) | 0-5 ms | Preemptive transmission |
| BLE transmission | 1-3 ms | Packet airtime |
| Belt reception | < 1 ms | Processing |
| Alert decision | 100-500 µs | Classification |
| Haptic trigger | < 1 ms | Actuator response |

### 16.3 Improvements for Extreme Velocity

| Improvement | Latency Reduction |
|-------------|-------------------|
| QoS = Critical preemption | 30 ms → 5 ms (worst case) |
| Reserved slot (25-28 ms) | Guaranteed transmission opportunity |
| Antenna diversity | Reduced retries from body shadowing |

**Net effect:** Total detection-to-alert latency reliably < 10 ms.

---

## 17. Testing and Quality Control

### 17.1 All Variants — Mandatory Tests

1. Power-on self-test — MCU boots, firmware CRC verified
2. Rail verification — all power rails within ±5% of nominal
3. BAN connectivity — ping all nodes, latency < 50 ms
4. WiFi connectivity — associate with test AP, latency < 100 ms
5. SD card detection — verify mount and read/write
6. IMU verification — gravity vector reads 9.81 ± 0.5 m/s² at rest
7. Environmental sensor — temperature, humidity read reasonable values

### 17.2 Linux SoM Variants — Additional Tests

8. Boot time — Linux boot complete within 30 seconds
9. API server — all basic REST endpoints return 200 OK
10. Media streaming — RTSP stream opens within 5 seconds
11. SLAM benchmark — 1000-frame PentaTrack update < 10 ms/frame
12. Thermal — sustained load for 1 hour, enclosure < 42°C

### 17.3 Cellular Module Tests (If Installed)

13. SIM detection — `SIM_DET` GPIO reads card present
14. SIM power — `SIM_VCC` within spec (1.8V or 3.0V)
15. AT command response — `AT` returns `OK` within 5 seconds
16. Network registration — module reports registered within 60 seconds (with live SIM)
17. Cellular current draw — within spec during active connection

### 17.4 Antenna Diversity Tests

18. Left antenna RSSI measurement — verify within expected range
19. Right antenna RSSI measurement — verify within expected range
20. Switching functionality — verify firmware switches antennas based on RSSI
21. Diversity gain — measure RSSI improvement with body shadowing

### 17.5 Hot-Swap Tests (Variant E Only)

22. Battery swap while running — system continues operation on remaining battery
23. Swap detection — firmware correctly reports which bay is active
24. Charge while operating — both bays charge while system runs

---

## 18. Directory Structure

```
belt_node/
├── variant_a_mcu/
│   ├── belt_node_mcu.kicad_pro
│   ├── belt_node_mcu.kicad_sch
│   ├── belt_node_mcu.kicad_pcb
│   └── gerbers/
├── variant_b_linux_som/
│   ├── belt_node_cm4.kicad_pro
│   ├── belt_node_cm4.kicad_sch
│   ├── belt_node_cm4.kicad_pcb
│   └── gerbers/
├── variant_c_imx8/
│   ├── belt_node_imx8.kicad_pro
│   └── gerbers/
├── variant_d_esp32/
│   ├── belt_node_esp32.kicad_pro
│   └── gerbers/
├── variant_e_hotswap/
│   ├── belt_node_hotswap.kicad_pro
│   └── gerbers/
├── bom/
│   ├── belt_node_bom_mcu.csv
│   ├── belt_node_bom_linux.csv
│   └── belt_node_bom_hotswap.csv
├── cad/
│   └── belt_enclosure.step
└── README.md
```

---

## 19. Configuration Summary

```toml
# sentinel-wear.toml — Belt Node Configuration

[identity]
node_type = "belt"
node_id = "belt_primary"

[compute]
variant = "linux_som"              # "mcu" | "linux_som" | "high_perf" | "esp32"
slam_enabled = true
slam_backend = "orb_slam3"          # "orb_slam3" | "lio_sam"

[connectivity]
# WiFi configuration
wifi_enabled = true
wifi_prefer_5ghz = true

# Cellular configuration
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false

# BAN configuration
[connectivity.ban]
primary = "ble"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

# BLE scheduling
[ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100
sleep_interval_ms = 500
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28

# Dynamic scheduling
[ban.scheduling]
mode = "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true

# QoS classes
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
traffic_types = ["presence_detection", "tracking_update", "identification"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "diagnostics"]

# Antenna diversity
[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
hysteresis_db = 2
preferred_antenna = "auto"

# UWB
[ban.uwb]
enabled = false
role = "timing_and_ranging"
nodes_with_uwb = ["pendant"]
streaming_fallback = false
max_sustained_mbps = 5

# RF coexistence
[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false

# Power management
[power]
battery_capacity_mah = 5000
low_battery_threshold_percent = 20
critical_battery_threshold_percent = 10

[power.battery_aware]
enabled = true
scale_slam_on_low_battery = true
scale_streaming_on_low_battery = true

[power.thermal]
max_enclosure_temp_c = 45
throttle_slam_above_temp_c = 40
disable_slam_above_temp_c = 45

# Sensors
[sensors.imu]
model = "ICM-42688-P"
sample_rate_hz = 200
full_scale_g = 16
low_noise_mode = true

[sensors.radar]
model = "IWR6843"
mode = "standard"

[sensors.radar.extended_doppler]
enabled = false
max_velocity_ms = 300
range_resolution_m = 0.5

# Storage
[storage]
primary = "sd_card"
retention_days = 30
max_storage_mb = 0
integrity_chain = true

# App server
[app]
enabled = true
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = ""
```
