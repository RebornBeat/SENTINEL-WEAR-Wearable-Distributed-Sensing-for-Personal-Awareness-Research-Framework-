# Connectivity Configuration Guide — SENTINEL-WEAR

**Version:** 1.0
**Status:** Complete Reference
**Applies To:** All node types, all variants

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [The Belt Node as Sole External Gateway](#2-the-belt-node-as-sole-external-gateway)
3. [Transport Layer Hierarchy](#3-transport-layer-hierarchy)
4. [Body-Area Network (BAN) Architecture](#4-body-area-network-ban-architecture)
5. [BLE Configuration and Scheduling](#5-ble-configuration-and-scheduling)
6. [UWB Configuration](#6-uwb-configuration)
7. [WiFi Configuration](#7-wifi-configuration)
8. [Cellular / SIM Configuration](#8-cellular--sim-configuration)
9. [RF Coexistence](#9-rf-coexistence)
10. [Antenna Diversity](#10-antenna-diversity)
11. [Extreme Velocity Detection Connectivity](#11-extreme-velocity-detection-connectivity)
12. [Configuration File Reference](#12-configuration-file-reference)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Architecture Overview

### 1.1 The Single Gateway Principle

SENTINEL-WEAR uses a **single gateway architecture** where only the belt node connects to external networks. All other nodes communicate exclusively via Body-Area Network (BAN) to the belt node.

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
│  ⚠️  ALL OTHER NODES = BAN ONLY (BLE/UWB to belt, nothing external)         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Why This Architecture

| Constraint | WiFi on Wearable | BLE on Wearable | UWB on Wearable |
|------------|------------------|-----------------|-----------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Form factor impact** | Requires larger battery, antenna space | Minimal | Moderate |
| **Wearability** | Breaks jewelry form factor | Preserves jewelry form | Compatible |
| **Determinism** | Variable latency, contention | Consistent, schedulable | Excellent timing |
| **Antenna** | Requires significant clearance | Small PCB antenna | Moderate clearance |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power/thermal/size requirements. BLE and UWB are the only viable BAN technologies for jewelry-scale wearables. All external connectivity (WiFi, Cellular) is concentrated at the belt node where larger form factor and battery are acceptable.

### 1.3 Data Flow Summary

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         DATA FLOW ARCHITECTURE                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SENSING NODES (Pendant, Bracelets, Anklets, Eyewear)                       │
│  │                                                                           │
│  ├── Sensor Data → MCU Processing → BAN Radio (BLE/UWB)                     │
│  │                                           │                               │
│  │                                           ▼                               │
│  │                                    [ BAN TRANSPORT ]                      │
│  │                                           │                               │
│  │                                           ▼                               │
│  └────────────────────────────────────► BELT NODE                            │
│                                              │                               │
│                                              ├──► Sparse Tracking (always)  │
│                                              ├──► Dense SLAM (if enabled)   │
│                                              ├──► Recording Manager          │
│                                              └──► Alert Routing             │
│                                                      │                       │
│                                                      ▼                       │
│                                              [ EXTERNAL GATEWAY ]            │
│                                                      │                       │
│                                    ┌─────────────────┼─────────────────┐     │
│                                    │                 │                 │     │
│                                    ▼                 ▼                 ▼     │
│                              WiFi (primary)    Cellular (remote)   BLE direct│
│                                    │                 │                 │     │
│                                    ▼                 ▼                 ▼     │
│                            Companion App      Remote App         Local App   │
│                            (full capability)  (alerts/streaming) (limited)   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The Belt Node as Sole External Gateway

### 2.1 What This Means

**Only the belt node has:**
- WiFi interface (802.11 a/b/g/n/ac/ax)
- Cellular modem (LTE/5G with SIM)
- BLE 5.x radio capable of connecting to companion app

**All other nodes have:**
- BLE 5.x radio (mandatory) — connects to belt only
- UWB radio (optional) — connects to belt only
- NO WiFi
- NO cellular
- NO direct companion app connection

### 2.2 Why This Is Architectural, Not Configurable

This is enforced at multiple levels:

| Level | Enforcement |
|-------|-------------|
| **Hardware** | Non-belt PCBs have no WiFi/cellular footprints |
| **Firmware** | Non-belt firmware binaries have no WiFi/cellular drivers |
| **Protocol** | Mesh protocol only defines BAN messages; no external routing |
| **Configuration** | No configuration option can enable external connectivity on non-belt nodes |

**Users cannot accidentally or intentionally configure a non-belt node to connect externally.**

### 2.3 User Responsibility

The belt node's external connectivity is fully user-configured:
- WiFi network selection
- Cellular SIM and carrier selection
- Remote access enable/disable
- Authentication tokens
- TLS/encryption settings

**No external connectivity is required for basic operation.** The system functions fully offline.

---

## 3. Transport Layer Hierarchy

### 3.1 Complete Hierarchy

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

### 3.2 Use Case to Transport Mapping

| Data Type | Primary Transport | Fallback | Notes |
|-----------|------------------|----------|-------|
| Control commands | BLE | — | Always via BAN |
| Health telemetry | BLE | — | Always via BAN |
| Detection metadata | BLE | — | Always via BAN |
| Gait events | BLE | — | Always via BAN |
| IMU data (streaming) | BLE | UWB burst | Bandwidth-dependent |
| Single camera stream | UWB | WiFi relay | UWB sufficient for 720p-1080p |
| 360° at 2K | UWB | WiFi relay | At UWB threshold |
| 360° at 4K | WiFi only | — | Exceeds UWB |
| SLAM data | WiFi | — | High bandwidth |
| Remote alerts | Cellular | — | When WiFi unavailable |
| Remote streaming | Cellular (5G) | — | Requires high bandwidth |

### 3.3 UWB Detailed Capabilities

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

### 3.4 Tiered Progressive Quality Strategy

When bandwidth is constrained (e.g., UWB streaming without WiFi):

```
Phase 1 (Instant — within 1 second):
    Send all 8 cameras at QVGA (320×240) MJPEG
    Total bandwidth: ~500 Kbps
    Belt receives and stitches rough 360° panorama
    User sees: Low-res but complete 360° view

Phase 2 (Progressive — over 5-10 seconds):
    Send higher-resolution keyframes from each camera sequentially:
    - Camera 0: 720p keyframe → belt updates stitching
    - Camera 1: 720p keyframe → belt updates stitching
    - ... (repeat for all 8 cameras)
    Each keyframe: ~100-200 KB compressed
    Total additional bandwidth: ~1.6-3.2 Mbps when combined with baseline

Phase 3 (Continuous):
    Maintain QVGA baseline stream continuously
    Periodically send higher-res keyframes
    Final panorama quality approaches what 720p would produce

Phase 4 (Adaptive):
    Monitor link quality:
    - Good link → Push higher resolution
    - Degraded link → Maintain baseline
    - Burst available → Send high-res keyframe burst
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
max_bandwidth_mbps = 5.0                  # Never exceed this (UWB limit)
```

---

## 4. Body-Area Network (BAN) Architecture

### 4.1 BAN Physical Layer Options

| Transport | Frequency | Max Range | Power | Nodes Supported |
|-----------|-----------|-----------|-------|------------------|
| BLE 5.x | 2.4 GHz ISM | 10-30 m (body) | 5-15 mW | Unlimited (scheduled) |
| UWB | 3.1-10.6 GHz | 10-50 m | 50-150 mW | Unlimited (time-slotted) |
| BLE + UWB Hybrid | Both | Combined | 55-165 mW | Optimal performance |

### 4.2 BLE vs UWB Role Allocation

**BLE handles:**
- Control plane (configuration, commands)
- Health and status telemetry
- Detection metadata and events
- Low-bandwidth sensor data
- Time synchronization (via connection events)

**UWB handles:**
- Precision time synchronization (sub-nanosecond)
- Precision ranging (cm-level distance)
- Camera streams (single or 360°)
- Sensor data bursts
- Heavy bandwidth transfers

### 4.3 When to Enable UWB

| Scenario | UWB Recommended | Reason |
|----------|-----------------|--------|
| Basic presence awareness | No | BLE sufficient |
| Gait analysis with precise sync | Yes | Sub-ns timing improves gait phase accuracy |
| Single camera streaming | Yes | UWB handles 720p-1080p |
| 360° pendant active | Yes | Required for reasonable quality |
| Extreme velocity detection | Recommended | Faster burst capability |
| Maximum battery life | No | BLE only is most efficient |

### 4.4 Multi-Radio BLE Considered and Rejected

**Why we do NOT use multiple BLE radios:**

| Issue | Multi-Radio BLE | Single BLE + Antenna Diversity |
|-------|-----------------|-------------------------------|
| RF self-interference | High risk (desensitization) | No self-interference |
| Antenna coupling | Severe at wearable scale | Controlled via antenna separation |
| Power consumption | 5-15 mW per radio | 5-15 mW total |
| Firmware complexity | High (multi-coordinator sync) | Lower |
| PCB complexity | High (multiple RF paths) | Lower |
| Reliability | Can be worse due to interference | More predictable |

**Correct solution for improved BLE reliability:**

- Single BLE coordinator with antenna diversity
- Deterministic scheduling
- QoS prioritization

**Multi-radio BLE makes sense only at extreme scale (>20 nodes, room/building deployment), not for single-person wearable systems.**

---

## 5. BLE Configuration and Scheduling

### 5.1 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture, extreme velocity |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | **Standard operation (default)** |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### 5.2 Deterministic Time-Slotted Scheduling

**Problem:** If all nodes transmit simultaneously, BLE collisions and latency spikes occur.

**Solution:** Fixed time slots within each connection interval.

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
└── Slot 28-30 ms: Retransmits / margin
```

**Benefits:**
- Guaranteed latency bounds for each node
- No collisions
- Predictable power consumption
- Simple firmware implementation

### 5.3 Node Priority for Bandwidth Allocation

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

### 5.4 Dynamic Scheduling

Context-aware adjustment of scheduling:

| Context | Adjustment | Reason |
|---------|------------|--------|
| Walking detected | Prioritize anklets | Gait timing critical |
| Threat detected | Prioritize pendant | Sensing critical |
| Idle state | Extend intervals | Power saving |
| Streaming active | Shift bursts to UWB | Reduce BLE load |
| Extreme velocity alert | Preempt all other traffic | Immediate response required |

**Configuration:**

```toml
[ban.scheduling]
mode = "dynamic"                 # "static" | "dynamic"

[ban.scheduling.static]
interval_ms = 30

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5    # Anklet slots increase by 50%
threat_pendant_priority_boost = 2.0     # Pendant slots double during threat
idle_interval_multiplier = 2.0          # Intervals double when idle
streaming_shift_to_uwb = true           # Move heavy traffic to UWB

# Extreme velocity preemption
extreme_velocity_qos = "critical"
extreme_velocity_preempt = true
extreme_velocity_reserved_slot = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
```

### 5.5 QoS Classes

| QoS Class | Priority | Traffic Types | Behavior |
|-----------|----------|---------------|----------|
| Critical | Highest | Extreme velocity, fall detection, emergency | Preempts all other traffic, immediate transmit |
| High | Second | Gait anomaly, stumble precursor | Priority slot allocation |
| Medium | Third | Presence detection, tracking update | Normal scheduled slot |
| Low | Fourth | Battery status, health telemetry | Sent only when bandwidth available |

**Configuration:**

```toml
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 5
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry"]
```

---

## 6. UWB Configuration

### 6.1 Role Types

| Role | Purpose | Bandwidth Usage | Power |
|------|---------|-----------------|-------|
| `timing_only` | Sub-ns sync, ranging | Minimal | ~50 mW |
| `timing_and_ranging` | Sync + distance measurement | Occasional bursts | ~80 mW |
| `full_bandwidth` | Sync + sustained streaming | Up to 5-6 Mbps | ~100-150 mW |

### 6.2 Configuration by Node Type

| Node Type | UWB Role Recommendation |
|-----------|------------------------|
| Pendant (standard) | `timing_only` or disabled |
| Pendant (360°) | `full_bandwidth` when streaming |
| Bracelet | `timing_only` or disabled |
| Anklet | `timing_and_ranging` (gait sync) |
| Eyewear | `timing_only` or disabled |
| Belt | `timing_and_ranging` (as coordinator) |

### 6.3 UWB Configuration

```toml
[ban.uwb]
enabled = false
role = "timing_and_ranging"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[ban.uwb.timing]
time_sync_interval_ms = 100
ranging_interval_ms = 500
precision_ns = 0.5                    # Sub-nanosecond precision

[ban.uwb.bandwidth]
burst_mode = true                    # Short bursts for efficiency
burst_duration_ms = 100
burst_interval_ms = 1000

[ban.uwb.offload]
# Traffic to offload to UWB instead of BLE
camera_burst = true
radar_burst = true
audio_clip = true
slam_data = true
extreme_velocity_event_camera_burst = true
```

### 6.4 UWB for 360° Pendant Streaming

**Configuration for streaming 360° over UWB:**

```toml
[pendant_360.uwb_streaming]
enabled = true
target_bandwidth_mbps = 4.5          # Conservative, within UWB limit
stream_mode = "progressive"          # "progressive" | "fixed_resolution"

[pendant_360.uwb_streaming.progressive]
baseline_resolution = "qvga"         # Start here
baseline_bandwidth_mbps = 0.5
refinement_resolution = "720p"       # Improve to this
refinement_interval_ms = 500
adaptive_quality = true
link_quality_monitoring = true

[pendant_360.uwb_streaming.fixed_resolution]
resolution = "2k"                    # Fixed 2K panorama
bandwidth_mbps = 4.0
```

---

## 7. WiFi Configuration

### 7.1 WiFi Is Belt Node Only

WiFi connectivity exists only on the belt node. All streaming and high-bandwidth operations route through the belt node's WiFi connection.

### 7.2 Band Selection

**5 GHz band strongly recommended** to avoid 2.4 GHz conflict with BLE.

| Band | BLE Conflict | Recommendation |
|------|--------------|----------------|
| 2.4 GHz | Yes — same spectrum | Avoid if possible |
| 5 GHz | No — separate spectrum | **Recommended** |
| 6 GHz (WiFi 6E) | No — separate spectrum | Best if available |

### 7.3 WiFi Configuration

```toml
[connectivity.wifi]
enabled = true
prefer_5ghz = true                  # Strongly recommended
auto_connect = true
ssid = ""                           # Set via companion app for security
password = ""                       # Set via companion app for security

[connectivity.wifi.power]
sleep_when_idle = true
tx_power_percent = 80               # Reduce for thermal management

[connectivity.wifi.streaming]
max_concurrent_streams = 4
preferred_codec = "h264"            # "h264" | "h265"
max_resolution = "4k"               # "720p" | "1080p" | "2k" | "4k"
bitrate_mbps = 15                   # Per stream
```

### 7.4 WiFi for Remote Access

When remote access is enabled, the belt node's WiFi can connect to a home network with internet access, enabling remote companion app connection.

```toml
[connectivity.wifi.remote]
enabled = false
port_forwarding_required = true     # User must configure router
external_port = 8080
tls_enabled = true                  # Required for remote access
```

---

## 8. Cellular / SIM Configuration

### 8.1 Cellular Is Belt Node Only

Cellular connectivity exists only on the belt node. It provides remote access when WiFi is unavailable.

### 8.2 Supported Cellular Modules

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |
| Sierra Wireless RV55 | LTE/5G | Variable | USB | Industrial grade |

### 8.3 SIM Types

| SIM Type | Advantage | Disadvantage |
|----------|-----------|--------------|
| nano-SIM (physical) | Easy carrier swap | Physical access required |
| eSIM (embedded) | No physical swap needed | Requires eSIM-compatible carrier |

### 8.4 Cellular Configuration

```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"              # "nano_sim" | "esim"
module = "ec25"                    # Module identifier

[connectivity.cellular.network]
apn = ""                           # Carrier APN
auto_detect_apn = true             # Try to detect automatically

[connectivity.cellular.power]
sleep_when_idle = true
keep_alive_interval_minutes = 5   # Periodic wake for network registration

[connectivity.cellular.usage]
fallback_to_wifi = true            # Prefer WiFi when available
always_on_cellular = false         # Keep cellular active even with WiFi
alert_via_cellular = true          # Send critical alerts via cellular
stream_via_cellular = false        # Stream video over cellular (data cost)

[connectivity.cellular.emergency]
contact = ""                       # Phone/endpoint for emergency alerts
trigger = "manual"                 # "manual" | "fall_detected" | "critical_alert"
location_included = true           # Include GPS if available
```

### 8.5 Data Usage Considerations

| Activity | Data Usage | Notes |
|----------|------------|-------|
| Alerts (metadata only) | < 10 KB each | Minimal |
| Compressed video clip (10 s) | 1-5 MB | LTE Cat 1 sufficient |
| Live streaming (1 min) | 50-100 MB | LTE Cat 4 or 5G |
| 360° live streaming (1 min) | 200-500 MB | 5G recommended |

**Recommendation:** Use cellular for alerts and metadata by default. Enable streaming only when necessary and with awareness of data costs.

---

## 9. RF Coexistence

### 9.1 Frequency Bands in Use

| Radio | Frequency Band | Conflict Risk |
|-------|---------------|---------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz and 5 GHz | BLE (2.4 GHz only) |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz | None (separate band) |
| Cellular LTE | 700 MHz - 2.6 GHz | Possible adjacent to 2.4 GHz |
| Cellular 5G Sub-6 | 3.5 GHz | Possible adjacent to UWB |
| Cellular 5G mmWave | 24-47 GHz | None (separate band) |

### 9.2 Coexistence Strategies

**Strategy 1: WiFi Prefers 5 GHz Band**

Most modern routers support 5 GHz. BLE operates at 2.4 GHz — no conflict.

```toml
[rf_coexistence]
wifi_prefer_5ghz = true            # Strongly recommended
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
ble_wifi_timeshare = false         # Set true only if 2.4 GHz WiFi required
ble_gap_during_wifi_ms = 5         # Gap between BLE and WiFi if sharing
```

### 9.3 Configuration Summary

```toml
[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false
ble_gap_during_wifi_ms = 5
```

---

## 10. Antenna Diversity

### 10.1 Why Antenna Diversity Instead of Multiple Radios

**The key insight:** For wearable systems, a single BLE radio with dual antennas provides better reliability than multiple BLE radios.

| Approach | RF Self-Interference | Complexity | Power | Reliability |
|----------|---------------------|------------|-------|-------------|
| Multi-radio BLE | High risk | High | High | Can be worse |
| Single radio + antenna diversity | None | Low | Low | Better |

### 10.2 How Antenna Diversity Works

```
Belt Node BLE Radio
    │
    ├── Antenna Switch (SPDT)
    │       ├── Antenna 1 (left side of belt)
    │       └── Antenna 2 (right side of belt)
    │
    └── Radio selects best antenna dynamically based on RSSI
```

**Benefits:**
- Reduced packet loss from body shadowing
- Lower retries
- More deterministic latency
- Critical for extreme velocity detection reliability

### 10.3 Antenna Diversity Configuration

```toml
[ban.antenna_diversity]
# Belt node antenna diversity
enabled = true
antenna_count = 2
selection_mode = "rssi"            # "rssi" | "packet_loss" | "manual"
switch_threshold_db = 5            # Switch if other antenna is 5 dB better
preferred_antenna = "auto"         # "left" | "right" | "auto"
hysteresis_db = 2                  # Don't switch too frequently

# Per-node antenna diversity (if supported)
[nodes.bracelet_left.antenna_diversity]
enabled = false

[nodes.anklet_left.antenna_diversity]
enabled = false
```

### 10.4 Antenna Diversity Implementation

**Hardware:**
- All belt PCBs have dual-antenna footprint
- Population is a configuration choice (enabled/disabled)
- Antenna switch IC (e.g., Skyworks SKY13330)

**Firmware:**
- Monitor RSSI on both antennas
- Switch when one antenna is significantly better
- Apply hysteresis to avoid rapid switching
- Log antenna switches for diagnostics

---

## 11. Extreme Velocity Detection Connectivity

### 11.1 Latency Requirements

| Scenario | Engagement Distance | Flight Time | Detection Budget |
|----------|---------------------|--------------|------------------|
| Rifle (850 m/s) | 3 m | 3.5 ms | < 1 ms ideal |
| Rifle | 50 m | 58.8 ms | < 5 ms acceptable |
| Rifle | 100 m | 117.6 ms | < 10 ms acceptable |
| Handgun (350 m/s) | 3 m | 8.5 ms | < 2 ms ideal |
| Handgun | 10 m | 28.6 ms | < 5 ms acceptable |

### 11.2 Connectivity Optimization for Extreme Velocity

**BLE Configuration:**

```toml
[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
direct_isr_to_radio = true
zero_copy_transmission = true
max_latency_ms = 5
```

**UWB Configuration:**

```toml
[extreme_velocity.uwb]
enabled = true
event_camera_burst = true
radar_burst = false              # CW radar detection sufficient
latency_target_ms = 2
```

### 11.3 Detection Pipeline Latency

**Tier 1 — Reflex Trigger (CW Radar):**
- CW radar detection: 50-150 µs
- Local haptic trigger: 50-300 µs (piezo)
- **Total: 100-450 µs**

**Tier 2 — Direction Validation (Event Camera):**
- Event camera capture: 10-100 µs
- Streak analysis: 100-500 µs
- BLE transmission (QoS Critical): 300-800 µs
- Belt reception: 50-150 µs
- **Total: 460-1550 µs**

**Complete alert latency:**
- With piezo haptic: 0.65-2.0 ms
- With LRA haptic: 2.5-11.5 ms

### 11.4 Warning Time at Realistic Distances

| Distance | Projectile | Flight Time | Alert Latency | Warning Time |
|----------|------------|-------------|---------------|--------------|
| 50 m | Rifle (850 m/s) | 58.8 ms | 0.65-2.0 ms | **56.8-58.2 ms** |
| 100 m | Rifle | 117.6 ms | 0.65-2.0 ms | **115.6-117.0 ms** |
| 200 m | Rifle | 235.3 ms | 0.65-2.0 ms | **233.3-234.7 ms** |
| 10 m | Handgun (350 m/s) | 28.6 ms | 0.65-2.0 ms | **26.6-28.0 ms** |

**At realistic rifle distances (100+ m), warning time is 115+ ms — well within human reaction time.**

---

## 12. Configuration File Reference

### 12.1 Complete Connectivity Configuration

```toml
# sentinel-wear.toml
# Complete connectivity configuration reference

# =============================================================================
# CONNECTIVITY
# =============================================================================

[connectivity]
# WiFi is primary external connection
wifi_enabled = true
wifi_prefer_5ghz = true

# =============================================================================
# BAN CONFIGURATION
# =============================================================================

[ban]
primary = "ble"                    # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

# BLE Scheduling
[ban.scheduling]
mode = "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true
extreme_velocity_qos = "critical"
extreme_velocity_preempt = true
extreme_velocity_reserved_slot = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28

# QoS Classes
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 5
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor"]

[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update"]

[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry"]

# UWB Configuration
[ban.uwb]
enabled = false
role = "timing_and_ranging"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[ban.uwb.timing]
time_sync_interval_ms = 100
ranging_interval_ms = 500
precision_ns = 0.5

[ban.uwb.bandwidth]
burst_mode = true
burst_duration_ms = 100
burst_interval_ms = 1000

[ban.uwb.offload]
camera_burst = true
radar_burst = true
audio_clip = true
slam_data = true
extreme_velocity_event_camera_burst = true

# Antenna Diversity
[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
preferred_antenna = "auto"
hysteresis_db = 2

# =============================================================================
# WiFi CONFIGURATION (BELT NODE ONLY)
# =============================================================================

[connectivity.wifi]
enabled = true
prefer_5ghz = true
auto_connect = true
ssid = ""
password = ""

[connectivity.wifi.power]
sleep_when_idle = true
tx_power_percent = 80

[connectivity.wifi.streaming]
max_concurrent_streams = 4
preferred_codec = "h264"
max_resolution = "4k"
bitrate_mbps = 15

[connectivity.wifi.remote]
enabled = false
port_forwarding_required = true
external_port = 8080
tls_enabled = true

# =============================================================================
# CELLULAR CONFIGURATION (BELT NODE ONLY)
# =============================================================================

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
module = "ec25"

[connectivity.cellular.network]
apn = ""
auto_detect_apn = true

[connectivity.cellular.power]
sleep_when_idle = true
keep_alive_interval_minutes = 5

[connectivity.cellular.usage]
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false

[connectivity.cellular.emergency]
contact = ""
trigger = "manual"
location_included = true

# =============================================================================
# RF COEXISTENCE
# =============================================================================

[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false
ble_gap_during_wifi_ms = 5

# =============================================================================
# EXTREME VELOCITY DETECTION
# =============================================================================

[extreme_velocity]
enabled = false
mode = "production"

[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
direct_isr_to_radio = true
zero_copy_transmission = true
max_latency_ms = 5

[extreme_velocity.uwb]
enabled = true
event_camera_burst = true
radar_burst = false
latency_target_ms = 2

# =============================================================================
# 360° PENDANT PROGRESSIVE QUALITY
# =============================================================================

[pendant_360.progressive_quality]
enabled = true
baseline_resolution = "qvga"
refinement_resolution = "720p"
refinement_interval_ms = 500
adaptive_bandwidth = true
min_bandwidth_mbps = 1.0
max_bandwidth_mbps = 5.0

[pendant_360.uwb_streaming]
enabled = true
target_bandwidth_mbps = 4.5
stream_mode = "progressive"
```

---

## 13. Troubleshooting

### 13.1 BLE Connectivity Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| High latency (> 50 ms) | Connection interval too long | Reduce `ble_connection_interval_ms` |
| Packet loss | RF interference or body shadowing | Enable antenna diversity |
| Missed slots | Too many nodes for interval | Increase interval or reduce node count |
| Connection drops | Out of range or power issue | Check battery; verify < 10 m distance |

### 13.2 UWB Streaming Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| Low quality | Bandwidth insufficient | Use progressive quality mode |
| Artifacts in 360° | Camera sync issue | Verify FSYNC timing |
| Drops during motion | Motion blur rejection | Increase acceptable motion threshold |
| High power drain | UWB always active | Disable when not streaming |

### 13.3 WiFi Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| Slow streaming | 2.4 GHz band | Verify `prefer_5ghz = true` |
| Connection drops | Weak signal | Check antenna, reduce distance to router |
| Cannot access remotely | Port forwarding | Configure router for external port |

### 13.4 Cellular Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| No connection | SIM not detected | Verify SIM insertion |
| High latency | Poor signal | Check antenna, verify coverage |
| Data expensive | Streaming enabled | Disable `stream_via_cellular` |

### 13.5 Extreme Velocity Detection Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| No detection | CW radar not configured | Enable extreme velocity mode |
| High latency | BLE not optimized | Enable QoS Critical, reserved slot |
| False positives | Sensitivity too high | Increase `velocity_threshold_ms` |
| Missing close-range | Detection range limited | Configure for appropriate range |

---

## Appendix A: Node Connectivity Summary

| Node Type | BLE | UWB | WiFi | Cellular | Notes |
|-----------|-----|-----|------|----------|-------|
| Pendant (Standard) | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Pendant (360°) | ✅ Mandatory | ⚠️ Recommended | ❌ None | ❌ None | UWB for bandwidth |
| Bracelet | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Anklet | ✅ Mandatory | ⚠️ Recommended | ❌ None | ❌ None | UWB for gait sync |
| Eyewear | ✅ Mandatory | ⚠️ Optional | ❌ None | ❌ None | BAN to belt only |
| Belt | ✅ Mandatory | ⚠️ Optional | ✅ Mandatory | ⚠️ Optional | Sole external gateway |

---

## Appendix B: Bandwidth Capability Summary

| Transport | Max Sustained | Max Burst | Primary Use |
|-----------|---------------|-----------|-------------|
| BLE 5.x | ~500 Kbps | ~2 Mbps | Control, metadata |
| UWB | ~5 Mbps | ~6.8 Mbps | Timing, moderate streaming |
| WiFi | 50-300+ Mbps | N/A | High bandwidth |
| LTE Cat 1 | 5-10 Mbps | N/A | Alerts, metadata |
| LTE Cat 4 | 50-150 Mbps | N/A | Moderate streaming |
| 5G | 100-1000+ Mbps | N/A | Live 360° streaming |

---

## Appendix C: Human Timescale Reference

| Human Function | Latency Threshold | SENTINEL-WEAR Target |
|----------------|-------------------|----------------------|
| Haptic perception | 10-20 ms | Alert < 20 ms |
| Motion perception | 20-50 ms | Detection display < 50 ms |
| Gait phase timing | 10-30 ms | Gait alert < 30 ms |
| Visual awareness | 50-100 ms | Visual feedback < 100 ms |
| Simple reaction | ~200 ms | Warning time > 100 ms for deliberate response |

---

## Appendix D: Related Documentation

| Document | Purpose |
|----------|---------|
| `hardware/hardware_config.md` | Hardware interfaces and pin mappings |
| `docs/theory/ban_bandwidth_budget.md` | Detailed bandwidth analysis |
| `docs/theory/rf_coexistence.md` | RF interference mitigation strategies |
| `docs/theory/extreme_velocity_sensing.md` | Extreme velocity detection theory |
| `docs/theory/doppler_radar_config.md` | Radar configuration for high-velocity detection |
| `firmware/firmware.md` | Firmware architecture for connectivity drivers |

---

**End of Document**
