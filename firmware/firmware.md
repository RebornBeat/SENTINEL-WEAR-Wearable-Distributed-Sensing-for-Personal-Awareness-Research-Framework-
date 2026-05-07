# Firmware Architecture — SENTINEL-WEAR

**Version:** 0.2 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V), Linux (Belt Node SoM)

---

## 1. Purpose

This document specifies the firmware architecture for SENTINEL-WEAR's sensing nodes. The firmware runs on resource-constrained MCUs embedded in jewelry-form-factor wearables (pendant, bracelets, anklets, eyewear) and on Linux-class SoMs in the belt node.

The firmware is implemented in Rust using the `no_std` ecosystem for MCUs and standard Rust for Linux SoM belt nodes. It is responsible for:
- Driving sensors (mmWave radar, IMU, LiDAR/ToF, acoustic, environmental, event, cameras).
- Performing node-local processing to reduce BAN bandwidth.
- Power management (critical for 24–72 hour battery life).
- Thermal management (critical for wearables with continuous operation).
- **For all non-belt nodes:** communicating with the belt controller over BAN (BLE 5.x or UWB) only. No WiFi, no cellular.
- **For the belt node:** managing BAN hub, WiFi connectivity, optional cellular SIM, companion app API server, SLAM processing, and recording lifecycle.

**Scope Constraint:** This firmware is sensing-and-information only. No effectors. Data handling is user-configured without system-imposed restrictions.

---

## 2. Connectivity Architecture — The Belt as Sole External Gateway

### Critical Architectural Principle

**Only the Belt Node connects to external networks.** This is enforced at three levels:

1. **Hardware level:** Belt node PCB includes WiFi module, cellular module, and external antenna connectors. All other node PCBs lack these components entirely — no footprints, no traces, no possibility.

2. **Firmware level:** Belt node firmware includes WiFi and cellular drivers. All other node firmware binaries have no WiFi or cellular driver dependencies — `wifi.rs` and `cellular.rs` are not compiled into non-belt binaries.

3. **Protocol level:** The BAN protocol vocabulary has no message types for external network communication. All messages are either internal (node-to-belt) or control (belt-to-node). External routing decisions are made exclusively by the belt node.

### Connectivity Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FIRMWARE CONNECTIVITY MODEL                              │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  FIRMWARE ENFORCEMENT:                                                       │
│  • Non-belt firmware: NO wifi driver, NO cellular driver, NO external IP    │
│  • Belt firmware: BAN hub + WiFi + Cellular + API server + recording        │
│  • All external connectivity decisions made by belt firmware only           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Non-Belt Node Firmware Constraints

```rust
// This is structurally enforced in the build system
// Non-belt binaries CANNOT depend on wifi or cellular features
#[cfg(all(feature = "wifi", not(feature = "belt_node")))]
compile_error!("WiFi is belt-node-only and cannot be used on other nodes");

#[cfg(all(feature = "cellular", not(feature = "belt_node")))]
compile_error!("Cellular is belt-node-only and cannot be used on other nodes");
```

### Belt Node Firmware Capabilities

The belt node firmware is the only firmware variant that links against:

| Driver | Belt Only? | Purpose |
|--------|------------|---------|
| `wifi.rs` | ✅ Yes | Connect to home network, serve companion app API |
| `cellular.rs` | ✅ Yes | LTE/5G connectivity for remote access |
| `ban_hub.rs` | ✅ Yes (as master) | Coordinate all BAN communication |
| `antenna_diversity.rs` | ✅ Yes (recommended) | Dual antenna switching for reliability |
| `slam.rs` | ✅ Yes (Linux SoM) | Dense world model construction |
| `api_server.rs` | ✅ Yes (Linux SoM) | HTTP/WebSocket/RTSP server |

---

## 3. Bandwidth Hierarchy and Transport Layer Selection

### BAN Transport Capabilities

The firmware implements intelligent transport selection based on data requirements:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE 5.x (500 Kbps - 2 Mbps practical)                                       │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, IMU orientation, classification results)            │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on, lowest power (5-15 mW)                                         │
│  → Present on ALL nodes — mandatory                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  UWB (4-6 Mbps sustained, 6.8 Mbps peak)                                     │
│  → Precision time synchronization (sub-nanosecond)                          │
│  → Precision ranging (cm-level distance between nodes)                       │
│  → Moderate-bandwidth continuous (single camera at 720p-1080p)               │
│  → 360° at 2K-2.5K resolution (stitched)                                     │
│  → High-bandwidth BURSTS (compressed clips, progressive keyframes)           │
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
│  → LTE Cat 4: 50-150 Mbps — moderate streaming                              │
│  → 5G: 100-1000+ Mbps — suitable for 360° live relay                         │
│  → Used when WiFi unavailable (remote access)                               │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### UWB Correct Bandwidth Capabilities

**UWB CAN handle continuous streaming at appropriate resolutions:**

| Stream Configuration | Bandwidth Required | UWB Capability | Notes |
|---------------------|-------------------|----------------|-------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ **Easily handled** | Instant baseline for 360° |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ **Handled** | Standard quality |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ **Handled** | Good quality |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ⚠️ **At max** | High quality |
| Single camera 720p-1080p H.264 | ~2-4 Mbps | ✅ **Handled** | Individual stream |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ **Exceeds** | Requires WiFi |
| 8 cameras × 720p raw | ~50+ Mbps | ❌ **Exceeds** | Not feasible |

**Key insight:** UWB is NOT a limitation — it is a capability that supports most use cases at appropriate quality levels. Higher resolutions require WiFi, which is available at the belt node.

### Tiered Progressive Quality Strategy

When bandwidth is constrained (UWB or poor WiFi), the firmware implements progressive quality:

```
Phase 1 (Instant — within 1 second):
    • Transmit all 8 cameras at QVGA (~500 Kbps total)
    • Belt receives and stitches complete but low-res 360° panorama
    • User sees functional view immediately

Phase 2 (Progressive — over 5-10 seconds):
    • Sequentially transmit higher-resolution keyframes
    • Camera 0: 720p keyframe → belt updates stitching
    • Camera 1: 720p keyframe → belt updates stitching
    • ... (all 8 cameras)
    • Combined bandwidth: ~2-4 Mbps (within UWB capability)

Phase 3 (Continuous refinement):
    • Maintain QVGA baseline continuously
    • Periodically send high-res keyframes
    • Quality improves progressively over time

Result: Immediate functionality, progressive improvement
```

**Firmware implication:** The belt node firmware must handle multiple quality tiers and adapt based on available bandwidth.

---

## 4. BLE Scheduling and Determinism

### Problem: Concurrent Node Transmission

If all nodes transmit simultaneously over BLE, collisions and latency spikes occur. The firmware implements a time-slotted scheduling scheme to guarantee determinism.

### BLE Connection Event Structure

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

### Reserved Slot for Critical Alerts

**Slot 25-28 ms is reserved for critical alerts including extreme velocity detection.** This guarantees that critical messages can be transmitted within one connection interval regardless of other traffic.

### Priority-Based Bandwidth Allocation

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

### Connection Interval Tradeoffs

| Interval | Average Latency | Power per Node | Use Case |
|----------|-----------------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation |
| 100 ms | ~60 ms | 2-5 mW | Power saving |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### Dynamic Scheduling

The firmware implements context-aware dynamic scheduling that adjusts priorities based on current activity:

| Context | Schedule Adjustment |
|---------|---------------------|
| Walking detected | Prioritize anklets (gait timing critical) |
| Threat detected | Prioritize pendant (sensing critical) |
| Idle state | Extend intervals (power saving) |
| Streaming active | Shift heavy traffic to UWB |
| Extreme velocity alert | Preempt all other traffic; use reserved slot |

```rust
/// Dynamic scheduling configuration
#[derive(Clone, Debug)]
pub struct DynamicScheduleConfig {
    /// Base connection interval
    pub base_interval_ms: u16,
    /// Context-based priority boosts
    pub walking_anklet_boost: f32,
    pub threat_pendant_boost: f32,
    /// Idle mode multiplier
    pub idle_interval_multiplier: f32,
    /// Enable critical alert preemption
    pub critical_preemption: bool,
}

impl Default for DynamicScheduleConfig {
    fn default() -> Self {
        Self {
            base_interval_ms: 30,
            walking_anklet_boost: 1.5,
            threat_pendant_boost: 2.0,
            idle_interval_multiplier: 2.0,
            critical_preemption: true,
        }
    }
}
```

### Firmware Configuration

```rust
/// BLE scheduling configuration
#[derive(Clone, Debug)]
pub struct BleScheduleConfig {
    /// Connection interval in milliseconds
    pub connection_interval_ms: u16,
    /// Node priority ordering
    pub node_priority: [NodeId; 6],
    /// Slot allocation per node in microseconds
    pub slot_allocation_us: [(NodeId, u16); 6],
    /// Reserved slot for critical alerts (microseconds)
    pub critical_slot_start_us: u16,
    pub critical_slot_end_us: u16,
    /// Dynamic scheduling configuration
    pub dynamic: DynamicScheduleConfig,
}

impl Default for BleScheduleConfig {
    fn default() -> Self {
        Self {
            connection_interval_ms: 30,
            node_priority: [
                NodeId::Belt,
                NodeId::AnkletL,
                NodeId::AnkletR,
                NodeId::Pendant,
                NodeId::BraceletL,
                NodeId::BraceletR,
            ],
            slot_allocation_us: [
                (NodeId::Pendant, 3000),
                (NodeId::AnkletL, 3000),
                (NodeId::AnkletR, 3000),
                (NodeId::BraceletL, 3000),
                (NodeId::BraceletR, 3000),
                (NodeId::Eyewear, 2000),
            ],
            critical_slot_start_us: 25000,
            critical_slot_end_us: 28000,
            dynamic: DynamicScheduleConfig::default(),
        }
    }
}
```

---

## 5. QoS Classes — Traffic Prioritization

### 5.1 QoS Class Definitions

The firmware implements a four-tier QoS system for BAN traffic:

```rust
/// Quality of Service classes for BAN traffic
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord)]
pub enum QoSClass {
    /// Lowest priority - background telemetry
    Low,
    /// Normal priority - standard detections
    Medium,
    /// High priority - time-sensitive detections
    High,
    /// Critical priority - preempts other traffic
    Critical,
}

/// Traffic type to QoS mapping
impl From<&TrafficType> for QoSClass {
    fn from(traffic_type: &TrafficType) -> Self {
        match traffic_type {
            // Critical - immediate transmission, preempt other traffic
            TrafficType::ExtremeVelocityDetection => QoSClass::Critical,
            TrafficType::FallDetection => QoSClass::Critical,
            TrafficType::EmergencyAlert => QoSClass::Critical,
            
            // High - priority queue, sent in next available slot
            TrafficType::GaitAnomaly => QoSClass::High,
            TrafficType::StumblePrecursor => QoSClass::High,
            TrafficType::IntrusionAlert => QoSClass::High,
            
            // Medium - normal scheduled slot
            TrafficType::PresenceDetection => QoSClass::Medium,
            TrafficType::TrackingUpdate => QoSClass::Medium,
            TrafficType::AcousticEvent => QoSClass::Medium,
            
            // Low - sent only when bandwidth available
            TrafficType::BatteryStatus => QoSClass::Low,
            TrafficType::HealthTelemetry => QoSClass::Low,
            TrafficType::ConfigurationSync => QoSClass::Low,
        }
    }
}

#[derive(Clone, Copy, Debug)]
pub enum TrafficType {
    // Critical
    ExtremeVelocityDetection,
    FallDetection,
    EmergencyAlert,
    
    // High
    GaitAnomaly,
    StumblePrecursor,
    IntrusionAlert,
    
    // Medium
    PresenceDetection,
    TrackingUpdate,
    AcousticEvent,
    
    // Low
    BatteryStatus,
    HealthTelemetry,
    ConfigurationSync,
}
```

### 5.2 QoS Implementation

```rust
/// QoS-aware message queue
pub struct QosMessageQueue {
    critical_queue: VecDeque<BanMessage>,
    high_queue: VecDeque<BanMessage>,
    medium_queue: VecDeque<BanMessage>,
    low_queue: VecDeque<BanMessage>,
}

impl QosMessageQueue {
    /// Enqueue message with appropriate priority
    pub fn enqueue(&mut self, message: BanMessage) {
        let qos = message.qos_class();
        match qos {
            QoSClass::Critical => self.critical_queue.push_back(message),
            QoSClass::High => self.high_queue.push_back(message),
            QoSClass::Medium => self.medium_queue.push_back(message),
            QoSClass::Low => self.low_queue.push_back(message),
        }
    }
    
    /// Get next message to transmit
    /// Critical messages are always sent first, even if it means preempting
    pub fn get_next(&mut self, can_preempt: bool) -> Option<BanMessage> {
        // Critical messages preempt everything
        if let Some(msg) = self.critical_queue.pop_front() {
            return Some(msg);
        }
        
        // If preemption not allowed, check reserved slot timing
        if !can_preempt {
            return None;
        }
        
        // High priority next
        if let Some(msg) = self.high_queue.pop_front() {
            return Some(msg);
        }
        
        // Medium priority
        if let Some(msg) = self.medium_queue.pop_front() {
            return Some(msg);
        }
        
        // Low priority only if bandwidth available
        self.low_queue.pop_front()
    }
    
    /// Check if critical message is pending
    pub fn has_critical(&self) -> bool {
        !self.critical_queue.is_empty()
    }
    
    /// Get queue depths for monitoring
    pub fn get_depths(&self) -> (usize, usize, usize, usize) {
        (
            self.critical_queue.len(),
            self.high_queue.len(),
            self.medium_queue.len(),
            self.low_queue.len(),
        )
    }
}
```

### 5.3 Critical Preemption for Extreme Velocity

```rust
/// Handle critical alert with immediate transmission
impl BanRadio {
    /// Send critical alert, preempting other traffic
    pub fn send_critical_alert(&mut self, alert: AlertClass, details: AlertDetails) -> Result<(), BanError> {
        // Create critical message
        let message = BanMessage::Alert {
            timestamp_ms: self.get_timestamp_ms(),
            alert_class: alert,
            priority: AlertPriority::Critical,
            source_node: self.node_id,
            description: details.description,
        };
        
        // If we're in the middle of another transmission,
        // abort it and send critical immediately
        if self.is_transmitting() && self.current_message_qos() < QoSClass::Critical {
            self.abort_transmission();
        }
        
        // Use reserved critical slot if available
        // Otherwise transmit immediately on next BLE event
        self.transmit_immediate(message)
    }
}
```

### 5.4 QoS Configuration

```toml
[ban.qos]
# Enable QoS prioritization
enabled = true

# Critical class configuration
[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
use_reserved_slot = true
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

# High class configuration
[ban.qos.high]
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor", "intrusion_alert"]

# Medium class configuration
[ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update", "acoustic_event"]

# Low class configuration
[ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry", "config_sync"]
```

---

## 6. Antenna Diversity — Reliability Enhancement

### 6.1 Why Antenna Diversity

The human body is a significant RF absorber at 2.4 GHz:
- Torso absorption: 10-30 dB attenuation
- Arm position changes: 5-15 dB variation
- Orientation changes: 10-20 dB variation

Single antenna configurations suffer from:
- Body shadowing (blocked signal when body is between antenna and target)
- Fading (signal strength variation with movement)
- Higher packet loss (leading to retries and increased latency)

**Antenna diversity** uses two physically separated antennas and switches between them to select the one with better signal quality.

### 6.2 Antenna Diversity Driver

```rust
/// Antenna diversity driver for improved BLE reliability
pub struct AntennaDiversityDriver {
    /// RF switch for antenna selection
    switch: RfSwitch,
    /// Current active antenna
    current_antenna: Antenna,
    /// RSSI history for each antenna
    rssi_history: [[i16; 8]; 2],  // [antenna_left, antenna_right]
    /// Configuration
    config: AntennaDiversityConfig,
}

#[derive(Clone, Copy, Debug, PartialEq)]
pub enum Antenna {
    Left,
    Right,
}

#[derive(Clone, Debug)]
pub struct AntennaDiversityConfig {
    /// Minimum RSSI difference before switching (hysteresis)
    pub switch_threshold_db: i16,
    /// Number of samples to consider for decision
    pub history_depth: usize,
    /// Preferred antenna for initial selection
    pub preferred_antenna: Antenna,
    /// Enable diversity (can be disabled for power saving)
    pub enabled: bool,
}

impl Default for AntennaDiversityConfig {
    fn default() -> Self {
        Self {
            switch_threshold_db: 5,
            history_depth: 8,
            preferred_antenna: Antenna::Left,
            enabled: true,
        }
    }
}

impl AntennaDiversityDriver {
    /// Create new diversity driver
    pub fn new(switch: RfSwitch, config: AntennaDiversityConfig) -> Self {
        Self {
            switch,
            current_antenna: config.preferred_antenna,
            rssi_history: [[-100i16; 8]; 2],
            config,
        }
    }
    
    /// Record RSSI sample for current antenna
    pub fn record_rssi(&mut self, rssi_db: i16) {
        let idx = self.rssi_history[self.current_antenna as usize]
            .iter()
            .position(|&r| r == -100)
            .unwrap_or(0);
        
        self.rssi_history[self.current_antenna as usize][idx % 8] = rssi_db;
        
        // Check if we should switch antennas
        self.evaluate_switch();
    }
    
    /// Evaluate whether to switch antennas
    fn evaluate_switch(&mut self) {
        if !self.config.enabled {
            return;
        }
        
        // Calculate average RSSI for each antenna
        let avg_left = self.average_rssi(Antenna::Left);
        let avg_right = self.average_rssi(Antenna::Right);
        
        // Determine if other antenna is significantly better
        let (other_antenna, improvement) = match self.current_antenna {
            Antenna::Left => (Antenna::Right, avg_right - avg_left),
            Antenna::Right => (Antenna::Left, avg_left - avg_right),
        };
        
        if improvement > self.config.switch_threshold_db {
            self.switch_to(other_antenna);
        }
    }
    
    /// Switch to specified antenna
    fn switch_to(&mut self, antenna: Antenna) {
        self.switch.select(antenna);
        self.current_antenna = antenna;
    }
    
    /// Calculate average RSSI for an antenna
    fn average_rssi(&self, antenna: Antenna) -> i16 {
        let history = &self.rssi_history[antenna as usize];
        let valid_samples: Vec<i16> = history.iter().filter(|&&r| r > -100).copied().collect();
        
        if valid_samples.is_empty() {
            return -100;
        }
        
        valid_samples.iter().sum::<i16>() / valid_samples.len() as i16
    }
    
    /// Get current antenna
    pub fn current(&self) -> Antenna {
        self.current_antenna
    }
    
    /// Get signal quality metric (higher = better)
    pub fn signal_quality(&self) -> i16 {
        self.average_rssi(self.current_antenna)
    }
}
```

### 6.3 Integration with BLE Driver

```rust
/// BLE driver with antenna diversity support
pub struct BleDriverWithDiversity<RADIO: BleRadio, SWITCH: RfSwitch> {
    radio: RADIO,
    antenna_diversity: AntennaDiversityDriver,
}

impl<RADIO: BleRadio, SWITCH: RfSwitch> BleDriverWithDiversity<RADIO, SWITCH> {
    /// Transmit packet with antenna diversity
    pub fn transmit(&mut self, packet: &BlePacket) -> Result<(), BleError> {
        // Record RSSI from last received packet
        if let Some(rssi) = self.radio.last_rssi() {
            self.antenna_diversity.record_rssi(rssi);
        }
        
        // Transmit using current best antenna
        self.radio.transmit(packet)
    }
    
    /// Receive packet with antenna diversity
    pub fn receive(&mut self) -> Result<Option<BlePacket>, BleError> {
        let packet = self.radio.receive()?;
        
        if let Some(ref pkt) = packet {
            if let Some(rssi) = pkt.rssi {
                self.antenna_diversity.record_rssi(rssi);
            }
        }
        
        Ok(packet)
    }
}
```

### 6.4 Antenna Diversity Configuration

```toml
[ban.antenna_diversity]
# Enable antenna diversity (belt node recommended)
enabled = true

# Number of antennas (1 = single, 2 = diversity)
antenna_count = 2

# Switch threshold in dB (higher = less frequent switching)
switch_threshold_db = 5

# Preferred antenna for initial connection
preferred_antenna = "auto"    # "left" | "right" | "auto"

# Hysteresis to prevent rapid switching
hysteresis_db = 2
```

---

## 7. RF Coexistence Firmware

### 7.1 Coexistence Challenges

The belt node operates multiple radios in close proximity:
- BLE (2.4 GHz)
- WiFi (2.4 GHz and 5 GHz)
- Cellular (various bands)
- UWB (3.1-10.6 GHz, separate from 2.4 GHz)

Primary conflict: BLE and 2.4 GHz WiFi share the same ISM band.

### 7.2 Coexistence Strategies

```rust
/// RF coexistence manager for belt node
#[cfg(feature = "belt_node")]
pub struct RfCoexistenceManager {
    /// WiFi band preference
    wifi_prefer_5ghz: bool,
    /// BLE-WiFi time sharing enabled
    ble_wifi_timeshare: bool,
    /// BLE gap during WiFi transmission (ms)
    ble_gap_ms: u16,
    /// Current WiFi activity state
    wifi_active: bool,
}

#[cfg(feature = "belt_node")]
impl RfCoexistenceManager {
    /// Create coexistence manager
    pub fn new(config: RfCoexistenceConfig) -> Self {
        Self {
            wifi_prefer_5ghz: config.wifi_prefer_5ghz,
            ble_wifi_timeshare: config.ble_wifi_timeshare,
            ble_gap_ms: config.ble_gap_ms,
            wifi_active: false,
        }
    }
    
    /// Notify WiFi activity starting
    pub fn on_wifi_activity_start(&mut self, ble: &mut BleDriver) {
        self.wifi_active = true;
        
        if self.ble_wifi_timeshare {
            // Pause BLE during WiFi transmission
            ble.pause_transmission(self.ble_gap_ms);
        }
    }
    
    /// Notify WiFi activity ended
    pub fn on_wifi_activity_end(&mut self, ble: &mut BleDriver) {
        self.wifi_active = false;
        
        if self.ble_wifi_timeshare {
            ble.resume_transmission();
        }
    }
    
    /// Check if 5 GHz WiFi should be preferred
    pub fn should_prefer_5ghz(&self) -> bool {
        self.wifi_prefer_5ghz
    }
}

#[derive(Clone, Debug)]
pub struct RfCoexistenceConfig {
    pub wifi_prefer_5ghz: bool,
    pub ble_wifi_timeshare: bool,
    pub ble_gap_ms: u16,
}

impl Default for RfCoexistenceConfig {
    fn default() -> Self {
        Self {
            wifi_prefer_5ghz: true,
            ble_wifi_timeshare: false,
            ble_gap_ms: 5,
        }
    }
}
```

### 7.3 RF Coexistence Configuration

```toml
[rf_coexistence]
# Prefer 5 GHz WiFi to avoid 2.4 GHz conflict with BLE
wifi_prefer_5ghz = true

# Enable BLE-WiFi time sharing (only needed if 2.4 GHz WiFi must be used)
ble_wifi_timeshare = false

# Gap between BLE and WiFi when time sharing
ble_gap_during_wifi_ms = 5
```

---

## 8. Target Platforms

### MCU Targets (Non-Belt Nodes)

**Primary:**
- ARM Cortex-M4F (STM32L4, nRF52840)
- ARM Cortex-M7 (STM32H7, i.MX RT1060)
- ARM Cortex-M33 (nRF5340, STM32U5)
- RISC-V (ESP32-C6)

**Resource Profile (Minimum):**
- Flash: 512 KB (1 MB preferred)
- RAM: 256 KB (512 KB for camera nodes)
- Storage: microSD for recording-capable nodes

### Linux SoM Targets (Belt Node)

| SoM | Architecture | RAM | NPU | Suitable For |
|-----|--------------|-----|-----|--------------|
| Raspberry Pi CM4 | Cortex-A72 | 2-8 GB | No | Sparse + basic SLAM |
| NXP i.MX 8M Plus | Cortex-A53 + M7 | 2-8 GB | 2.3 TOPS | Full SLAM + streaming |
| Qualcomm SA8155P | Kryo 385 | 4-16 GB | GPU | Maximum performance |
| Jetson Orin NX | ARM + NVIDIA GPU | 8-32 GB | 70-100 TOPS | Production deployment |

---

## 9. Workspace Structure

```
firmware/
├── Cargo.toml
├── build.rs
├── memory.x
├── firmware.md
├── README.md
└── src/
    ├── lib.rs
    ├── bin/
    │   ├── pendant_node.rs         # Standard/Medallion (Variants A, C, D, E)
    │   ├── pendant_360.rs          # 360° Curved (Variant B)
    │   ├── bracelet_left.rs
    │   ├── bracelet_right.rs
    │   ├── belt_node.rs            # MCU variants (A, D)
    │   ├── belt_node_linux.rs      # Linux SoM variants (B, C, E)
    │   ├── anklet_left.rs
    │   ├── anklet_right.rs
    │   └── eyewear_node.rs
    ├── drivers/
    │   ├── mod.rs
    │   ├── mmwave.rs
    │   ├── mmwave_doppler.rs       # Extended Doppler for extreme velocity
    │   ├── imu.rs
    │   ├── acoustic.rs
    │   ├── lidar_tof.rs
    │   ├── event_sensor.rs
    │   ├── environmental.rs
    │   ├── haptic.rs
    │   ├── camera.rs               # Single camera
    │   ├── camera_360.rs           # 360° multi-camera
    │   ├── storage.rs              # SD card + integrity chain
    │   ├── wifi.rs                 # Belt node only
    │   ├── cellular.rs             # Belt node only
    │   ├── uwb.rs                  # Optional on all nodes
    │   └── antenna_diversity.rs    # Belt node recommended
    ├── logic/
    │   ├── mod.rs
    │   ├── body_frame_local.rs
    │   ├── presence_detection.rs
    │   ├── gait_analysis.rs        # On-node gait event detection
    │   ├── extreme_velocity.rs     # Doppler + event fusion
    │   ├── recording_manager.rs
    │   ├── power_management.rs
    │   ├── thermal_management.rs
    │   └── progressive_quality.rs  # Tiered quality
    └── ban_radio/
        ├── mod.rs
        ├── ble_scheduler.rs        # Time-slotted scheduling
        ├── ble_qos.rs              # QoS classes and preemption
        ├── uwb_driver.rs           # UWB timing/ranging/bandwidth
        ├── protocol.rs             # BAN message definitions
        └── rf_coexistence.rs       # Belt node RF coordination
```

---

## 10. Node Binary Descriptions

### 10.1 Belt Node — MCU Variant (`belt_node.rs`)

**Role:** Primary compute, BAN hub, **sole external network gateway**, API server.

**Drivers used:** `mmwave.rs`, `imu.rs`, `environmental.rs`, `storage.rs`, `wifi.rs`, `cellular.rs`, `antenna_diversity.rs`

**Firmware responsibilities:**
- **BAN hub:** Coordinates all inter-node communication, time-sync master, BLE scheduler with QoS
- **WiFi connectivity:** Connects to home network; serves minimal API server
- **Cellular SIM:** Maintains LTE connection when configured; forwards alerts
- **Recording manager:** Aggregates `RecordingAvailable` events from all nodes; manages belt SD storage
- **Antenna diversity:** Switches between left/right BLE antennas for reliability
- **RF coexistence:** Manages BLE-WiFi coordination

**Limitations (MCU variant):**
- No dense SLAM (insufficient compute)
- Limited camera stream handling (1-2 streams)
- Minimal API server (REST only, no WebSocket)

### 10.2 Belt Node — Linux SoM Variant (`belt_node_linux.rs`)

**Role:** Full capability belt node with SLAM, 360° processing, complete API server.

**Additional drivers:** `slam.rs`, `stitching.rs`, `api_server.rs`

**Additional responsibilities:**
- **SLAM processing:** Dense world model construction
- **360° stitching:** Real-time panorama from pendant camera array
- **Full API server:** REST + WebSocket + RTSP
- **Progressive quality:** Receives tiered streams from 360° pendant, reconstructs high quality
- **Thermal management:** Monitors enclosure temperature, throttles SLAM if needed

### 10.3 Pendant Node — Standard/Medallion (`pendant_node.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, `acoustic.rs`, `camera.rs` (optional), `storage.rs` (optional), `uwb.rs` (optional)

**Key behaviors:**
- Multi-sensor fusion on-node (reduce BAN bandwidth)
- Camera data handling per `sentinel-wear.toml`:
  - Metadata-only: inference on-MCU; transmit classification via BAN
  - Local storage: frames → SD card; `RecordingAvailable` → BAN → belt
  - Belt relay: encoded video → BAN → belt → WiFi/cellular → app
- Optional hardware kill switch: read GPIO at boot; if DISABLED, camera driver not initialized
- All data exits pendant via BAN only

**Power states:**
- Active (sensing + radio): 200-500 mW
- Idle (sensors active, radio low-power): 50-100 mW
- Sleep (minimal): <10 mW

### 10.4 Pendant Node — 360° Curved (`pendant_360.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, `acoustic.rs`, `camera_360.rs`, `storage.rs`, `uwb.rs`

**Unique responsibilities:**
- Initialize N camera modules simultaneously (N = 4, 6, or 8)
- Generate FSYNC hardware sync pulse to all cameras
- Load calibration JSON from flash at boot
- Implement tiered progressive quality:
  - Phase 1: QVGA baseline from all cameras (~500 Kbps)
  - Phase 2: Sequential high-res keyframes (~2-4 Mbps burst)
  - Phase 3: Continuous refinement
- Transmit via BAN (BLE or UWB) to belt
- Belt handles stitching and WiFi/cellular relay

**FSYNC implementation:**
```rust
/// Hardware-synced camera capture
pub struct Camera360Capture {
    cameras: [CameraDriver; MAX_CAMERAS],
    camera_count: usize,
    fsync_gpio: GpioPin<Output<PushPull>>,
    calibration: CalibrationData,
}

impl Camera360Capture {
    /// Capture all cameras simultaneously with hardware sync
    pub fn capture_sync(&mut self) -> Result<[Frame; MAX_CAMERAS], CameraError> {
        // Assert FSYNC - all cameras start exposure simultaneously
        self.fsync_gpio.set_high();
        
        // Wait for exposure time
        self.delay.delay_us(self.exposure_us);
        
        // Deassert FSYNC
        self.fsync_gpio.set_low();
        
        // Read frames from all cameras
        let mut frames = [Frame::empty(); MAX_CAMERAS];
        for (i, camera) in self.cameras.iter_mut().enumerate().take(self.camera_count) {
            frames[i] = camera.read_frame()?;
        }
        
        Ok(frames)
    }
}
```

**Progressive quality implementation:**
```rust
/// Tiered progressive quality configuration
#[derive(Clone, Copy, Debug)]
pub enum QualityTier {
    /// Instant baseline: QVGA from all cameras
    Baseline,
    /// Standard: VGA from all cameras
    Standard,
    /// High: 720p keyframes
    High,
    /// Progressive: QVGA baseline + high-res keyframes
    Progressive,
}

impl Camera360Capture {
    /// Encode with tiered quality
    pub fn encode_progressive(
        &mut self,
        frames: &[Frame],
        tier: QualityTier,
        available_bandwidth_bps: u32,
    ) -> Result<EncodedStream, EncodeError> {
        match tier {
            QualityTier::Baseline => {
                // All cameras at QVGA, MJPEG (~500 Kbps total)
                self.encode_all_at_resolution(Resolution::QVGA, Codec::MJPEG)
            }
            QualityTier::Standard => {
                // All cameras at VGA, H.264 (~2-4 Mbps total)
                self.encode_all_at_resolution(Resolution::VGA, Codec::H264)
            }
            QualityTier::High => {
                // Subset at 720p
                let cameras_to_encode = (available_bandwidth_bps / 500_000).min(8) as usize;
                self.encode_keyframes(cameras_to_encode, Resolution::P720, Codec::H264)
            }
            QualityTier::Progressive => {
                // Baseline + periodic refinements
                self.encode_continuous()
            }
        }
    }
}
```

### 10.5 Pendant Node — Event-Enhanced (Variant E)

**Additional drivers:** `mmwave_doppler.rs`, `event_sensor.rs`

**Unique responsibilities:**
- Continuous Doppler radar monitoring in extended-velocity mode
- Event camera always-on for microsecond-latency detection
- Fusion of Doppler + event for extreme velocity detection
- Trigger high-res capture from conventional cameras when fast object detected

```rust
/// Extreme velocity detection loop
pub fn extreme_velocity_loop(
    doppler: &mut DopplerRadarDriver,
    event_cam: &mut EventSensorDriver,
    conventional_cams: &mut Camera360Capture,
    ban: &mut BanRadio,
) -> ! {
    loop {
        // Check Doppler for high-velocity returns
        if let Some(detection) = doppler.poll_high_velocity() {
            if detection.velocity_ms > 50.0 {
                // Trigger event camera capture
                if let Some(event_track) = event_cam.poll_fast_transient() {
                    // Confirm with event camera trajectory
                    if event_track.confidence > 0.8 {
                        // Create critical alert (QoS = Critical)
                        let alert = BanMessage::FastObjectDetection {
                            timestamp_us: detection.timestamp_us,
                            node_id: ban.node_id(),
                            velocity_ms: detection.velocity_ms,
                            bearing_rad: detection.bearing_rad,
                            elevation_rad: detection.elevation_rad,
                            range_m: detection.range_m,
                            confidence: detection.confidence,
                        };
                        
                        // Send with QoS Critical - preempts other traffic
                        ban.send_critical(alert)?;
                        
                        // Capture high-res from conventional cameras
                        let frames = conventional_cams.capture_sync()?;
                        ban.send_frames_high_priority(&frames);
                    }
                }
            }
        }
        
        // Normal sensing loop continues
        // ...
    }
}
```

### 10.6 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, `haptic.rs`, optional `camera.rs`, optional `uwb.rs`

**Key behaviors:**
- Outward-facing radar (dorsal side of wrist)
- Directional haptic encoding
- Optional forward-facing camera
- All data → BAN → belt

**Haptic alert encoding:**
```rust
/// Directional haptic encoding
pub fn encode_haptic_direction(
    approach_bearing_rad: f32,
    approach_elevation_rad: f32,
) -> HapticPattern {
    // Determine which bracelet should fire based on approach direction
    // Left bracelet: bearing ~PI to 2PI (left side)
    // Right bracelet: bearing ~0 to PI (right side)
    
    let is_left_side = approach_bearing_rad > std::f32::consts::FRAC_PI_2 
        && approach_bearing_rad < 1.5 * std::f32::consts::PI;
    
    if is_left_side {
        HapticPattern::LeftWrist {
            intensity: calculate_intensity(approach_elevation_rad),
            pattern: PulsePattern::Double,
        }
    } else {
        HapticPattern::RightWrist {
            intensity: calculate_intensity(approach_elevation_rad),
            pattern: PulsePattern::Double,
        }
    }
}
```

### 10.7 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

**Drivers:** `lidar_tof.rs`, `imu.rs`, `haptic.rs`, optional `camera.rs`, `uwb.rs` (recommended)

**Key behaviors:**
- Ground-plane sensing with ToF/LiDAR
- High-dynamic-range IMU for heel-strike (5-10 g range required)
- On-node gait event detection (transmit events, not raw IMU)
- Raw IMU burst on anomaly detection
- UWB recommended for precise gait synchronization between left/right anklets

**Gait analysis on-node:**
```rust
/// Gait event detection - runs on anklet MCU
pub struct GaitAnalyzer {
    imu: ImuDriver,
    heel_strike_threshold_g: f32,
    stride_samples: CircularBuffer<ImuSample, 200>,
}

impl GaitAnalyzer {
    /// Process IMU sample, detect gait events
    pub fn process_sample(&mut self, sample: ImuSample) -> Option<GaitEvent> {
        self.stride_samples.push(sample);
        
        // Detect heel-strike
        let accel_magnitude = sample.acceleration.norm();
        if accel_magnitude > self.heel_strike_threshold_g {
            let gait_phase = self.detect_phase(&sample);
            
            // Return event, not raw sample
            return Some(GaitEvent {
                event_type: GaitEventType::HeelStrike,
                phase: gait_phase,
                impact_g: (accel_magnitude * 10.0) as u8,
                timestamp_ms: sample.timestamp_ms,
            });
        }
        
        None
    }
    
    /// Detect pre-stumble signature
    pub fn detect_stumble_risk(&self) -> Option<StumblePrecursor> {
        // Analyze recent stride patterns for irregularity
        // ...
        None
    }
}
```

### 10.8 Eyewear Node (`eyewear_node.rs`)

**Drivers:** `event_sensor.rs`, `imu.rs`, optional `camera.rs`, optional `uwb.rs`

**Key behaviors:**
- Head-stabilized forward sensing
- Event camera always-on for fast transient detection
- Optional conventional camera for visual context
- Separate head orientation from torso orientation

**Head-torso separation:**
```rust
/// Maintain separate head orientation
pub struct HeadOrientationTracker {
    imu: ImuDriver,
    // Received from belt via BAN
    torso_quaternion: [f32; 4],
    head_to_torso_quaternion: [f32; 4],
}

impl HeadOrientationTracker {
    /// Update head orientation and compute relative to torso
    pub fn update(&mut self, imu_sample: &ImuSample) {
        let head_quaternion = self.imu.compute_quaternion(imu_sample);
        
        // Compute relative orientation (where is head looking vs torso facing)
        self.head_to_torso_quaternion = quaternion_relative(
            head_quaternion,
            self.torso_quaternion,
        );
    }
    
    /// Transform a detection from head-frame to torso-frame
    pub fn transform_detection_to_torso(&self, detection: &Detection) -> Detection {
        // Apply head-to-torso transform
        // ...
        detection.clone()
    }
}
```

---

## 11. Driver Layer

### 11.1 `mmwave.rs` — Standard mmWave Radar

```rust
/// mmWave radar driver for presence detection
pub struct MmWaveDriver<SPI: SpiDevice> {
    spi: SPI,
    config: MmWaveConfig,
}

impl<SPI: SpiDevice> MmWaveDriver<SPI> {
    pub fn init(&mut self) -> Result<(), RadarError>;
    pub fn scan(&mut self) -> Result<RadarScan, RadarError>;
    pub fn get_micro_doppler(&mut self) -> Result<MicroDopplerSignature, RadarError>;
}

pub struct RadarScan {
    pub detections: Vec<RadarDetection>,
    pub timestamp_ms: u64,
}

pub struct RadarDetection {
    pub range_m: f32,
    pub bearing_rad: f32,
    pub elevation_rad: f32,
    pub velocity_ms: f32,
    pub snr_db: f32,
}
```

### 11.2 `mmwave_doppler.rs` — Extended Doppler for Extreme Velocity

```rust
/// Doppler radar driver configured for extreme velocity detection
pub struct DopplerRadarDriver<SPI: SpiDevice> {
    spi: SPI,
    config: DopplerConfig,
    max_velocity_ms: f32,
}

#[derive(Clone, Copy, Debug)]
pub struct DopplerConfig {
    /// Maximum velocity to detect (m/s)
    /// Standard: ~18 m/s, Extended: 300+ m/s
    pub max_velocity_ms: f32,
    /// Range resolution tradeoff
    pub range_resolution_m: f32,
    /// Update rate (Hz) - higher for fast transients
    pub update_rate_hz: u32,
    /// CW mode for zero latency
    pub cw_mode: bool,
}

impl<SPI: SpiDevice> DopplerRadarDriver<SPI> {
    /// Configure for extended velocity range
    pub fn configure_extended(&mut self, max_velocity_ms: f32) -> Result<(), RadarError>;
    
    /// Poll for high-velocity detection
    pub fn poll_high_velocity(&mut self) -> Option<HighVelocityDetection>;
}

pub struct HighVelocityDetection {
    pub velocity_ms: f32,
    pub bearing_rad: f32,
    pub range_m: f32,
    pub confidence: f32,
    pub timestamp_us: u64,
}
```

### 11.3 `camera_360.rs` — 360° Multi-Camera Array

```rust
/// 360° camera array driver
pub struct Camera360Driver {
    cameras: [CameraDriver; MAX_CAMERAS],
    camera_count: usize,
    fsync_gpio: GpioPin<Output<PushPull>>,
    calibration: CalibrationData,
}

impl Camera360Driver {
    pub fn init(&mut self) -> Result<(), CameraError>;
    pub fn capture_sync(&mut self) -> Result<[Frame; MAX_CAMERAS], CameraError>;
    pub fn load_calibration(&mut self) -> Result<CalibrationData, StorageError>;
    pub fn encode_tiered(
        &mut self,
        frames: &[Frame],
        tier: QualityTier,
        bandwidth_bps: u32,
    ) -> Result<EncodedStream, EncodeError>;
}

#[derive(Clone, Copy, Debug)]
pub enum QualityTier {
    Baseline,    // QVGA all cameras
    Standard,    // VGA all cameras
    High,        // 720p keyframes
    Progressive, // QVGA + progressive refinement
}
```

### 11.4 `wifi.rs` — Belt Node Only

```rust
#[cfg(feature = "belt_node")]
pub struct WifiDriver {
    #[cfg(target_os = "linux")]
    interface: String,
}

#[cfg(feature = "belt_node")]
impl WifiDriver {
    pub fn connect(&mut self, ssid: &str, password: &str) -> Result<(), WifiError>;
    pub fn is_connected(&self) -> bool;
    pub fn get_local_ip(&self) -> Option<Ipv4Addr>;
    pub fn get_rssi(&self) -> Option<i32>;
    pub fn prefer_5ghz(&mut self, prefer: bool);
}
```

### 11.5 `cellular.rs` — Belt Node Only

```rust
#[cfg(feature = "belt_node")]
pub struct CellularDriver<UART: SerialPort> {
    uart: UART,
    config: CellularConfig,
    registered: bool,
}

#[cfg(feature = "belt_node")]
impl<UART: SerialPort> CellularDriver<UART> {
    pub fn power_on(&mut self) -> Result<(), CellularError>;
    pub fn register(&mut self) -> Result<(), CellularError>;
    pub fn is_registered(&self) -> bool;
    pub fn get_signal_strength(&mut self) -> Result<i32, CellularError>;
    pub fn get_carrier(&mut self) -> Result<String, CellularError>;
    pub fn send_at(&mut self, cmd: &str) -> Result<String, CellularError>;
    pub fn send_alert(&mut self, endpoint: &str, payload: &[u8]) -> Result<(), CellularError>;
    pub fn check_sim(&mut self) -> Result<SimStatus, CellularError>;
}
```

### 11.6 `uwb.rs` — Optional on All Nodes

```rust
pub struct UwbDriver<SPI: SpiDevice> {
    spi: SPI,
    config: UwbConfig,
    role: UwbRole,
}

#[derive(Clone, Copy, Debug)]
pub enum UwbRole {
    TimingOnly,
    TimingAndRanging,
    FullBandwidth,
}

impl<SPI: SpiDevice> UwbDriver<SPI> {
    pub fn init(&mut self) -> Result<(), UwbError>;
    pub fn time_sync_exchange(&mut self) -> Result<TimeSyncResult, UwbError>;
    pub fn measure_range(&mut self, target: NodeId) -> Result<f32, UwbError>;
    pub fn send_burst(&mut self, data: &[u8]) -> Result<(), UwbError>;
    pub fn start_stream(&mut self) -> Result<(), UwbError>;
    pub fn check_bandwidth(&self) -> f32;
}
```

### 11.7 `antenna_diversity.rs` — Belt Node Recommended

```rust
pub struct AntennaDiversityDriver {
    switch: RfSwitch,
    current_antenna: Antenna,
    rssi_history: [[i16; 8]; 2],
    config: AntennaDiversityConfig,
}

#[derive(Clone, Copy, Debug, PartialEq)]
pub enum Antenna {
    Left,
    Right,
}

impl AntennaDiversityDriver {
    pub fn new(switch: RfSwitch, config: AntennaDiversityConfig) -> Self;
    pub fn record_rssi(&mut self, rssi_db: i16);
    pub fn current(&self) -> Antenna;
    pub fn signal_quality(&self) -> i16;
}
```

### 11.8 `storage.rs` — Any Node With SD Card

```rust
pub struct StorageDriver<SPI: SpiDevice> {
    spi: SPI,
    sd_card: SdCard,
    config: StorageConfig,
}

impl<SPI: SpiDevice> StorageDriver<SPI> {
    pub fn init(&mut self) -> Result<(), StorageError>;
    pub fn open_recording(&mut self, path: &str) -> Result<RecordingWriter, StorageError>;
    pub fn list_recordings(&self) -> Result<Vec<RecordingMetadata>, StorageError>;
    pub fn delete_recording(&mut self, path: &str) -> Result<(), StorageError>;
    pub fn enforce_retention(&mut self) -> Result<u64, StorageError>;
    pub fn get_used_mb(&self) -> u64;
}

pub struct RecordingWriter<'a> {
    file: File<'a>,
    hasher: Sha256,
    start_time_ms: u64,
}

impl<'a> RecordingWriter<'a> {
    pub fn write_frame(&mut self, frame: &[u8]) -> Result<(), StorageError>;
    pub fn close(self) -> Result<RecordingMetadata, StorageError>;
}
```

---

## 12. BAN Protocol Stack

### 12.1 Protocol Overview

All inter-node communication uses the Body-Area Network (BAN) protocol. External connectivity (WiFi, cellular) is handled exclusively by the belt node.

### 12.2 Message Definitions

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum SentinelBanMessage {
    NodeSensorData {
        timestamp_ms: u64,
        node_id: NodeId,
        radar_metadata: RadarMetadata,
        imu_quaternion: [f32; 4],
        linear_accel: [f32; 3],
        audio_clip_available: bool,
        audio_clip_path: Option<String>,
    },
    
    Detection {
        timestamp_ms: u64,
        node_id: NodeId,
        detection_type: DetectionType,
        position: Vector3,
        velocity: Vector3,
        confidence: f32,
        classification: ClassificationHint,
    },
    
    GaitEvent {
        timestamp_ms: u64,
        node_id: NodeId,
        event_type: GaitEventType,
        phase: GaitPhase,
        confidence: f32,
        impact_g: Option<u8>,
    },
    
    AcousticEvent {
        timestamp_ms: u64,
        node_id: NodeId,
        event_class: String,
        direction_of_arrival: (f32, f32),
        confidence: f32,
        audio_clip_path: Option<String>,
    },
    
    IdentificationResult {
        timestamp_ms: u64,
        node_id: NodeId,
        classification: ClassificationTag,
        confidence: f32,
        raw_media_path: Option<String>,
        live_stream_available: bool,
        clip_duration_s: Option<f32>,
    },
    
    FastObjectDetection {
        timestamp_us: u64,
        node_id: NodeId,
        velocity_ms: f32,
        bearing_rad: f32,
        elevation_rad: f32,
        range_m: f32,
        confidence: f32,
    },
    
    RecordingAvailable {
        node_id: NodeId,
        recording_path: String,
        start_timestamp_ms: u64,
        duration_ms: u32,
        size_bytes: u64,
        trigger: String,
        format: String,
        modalities: Vec<String>,
        sha256: String,
    },
    
    StartStreamToBelt {
        node_id: NodeId,
        format: StreamFormat,
        quality: StreamQuality,
        use_uwb: bool,
    },
    
    StopStreamToBelt {
        node_id: NodeId,
    },
    
    StreamData {
        node_id: NodeId,
        timestamp_ms: u64,
        frame_number: u32,
        data: Vec<u8>,
    },
    
    PanoramaFrameAvailable {
        node_id: NodeId,
        timestamp_ms: u64,
        width: u32,
        height: u32,
        frame_path: Option<String>,
        quality_tier: QualityTier,
    },
    
    Alert {
        timestamp_ms: u64,
        alert_class: AlertClass,
        priority: AlertPriority,
        source_node: NodeId,
        description: String,
    },
    
    HapticCommand {
        target_node: NodeId,
        pattern: HapticPattern,
        duration_ms: u16,
        intensity: u8,
    },
    
    CalibrationCommand {
        command: CalibrationPhase,
    },
    
    TimeSyncExchange {
        t1_ns: u64,
        t2_ns: u64,
        t3_ns: u64,
        t4_ns: u64,
    },
    
    NodeHealth {
        node_id: NodeId,
        battery_percent: u8,
        battery_mv: u16,
        charging: bool,
        temperature_c: f32,
        sensor_health: Vec<(String, SensorHealthStatus)>,
    },
    
    ConfigUpdate {
        node_id: NodeId,
        config_section: String,
        config_data: Vec<u8>,
    },
}
```

### 12.3 QoS Integration in Protocol

```rust
impl SentinelBanMessage {
    /// Get QoS class for this message
    pub fn qos_class(&self) -> QoSClass {
        match self {
            Self::FastObjectDetection { .. } => QoSClass::Critical,
            Self::Alert { priority: AlertPriority::Critical, .. } => QoSClass::Critical,
            Self::GaitEvent { event_type: GaitEventType::FallDetected, .. } => QoSClass::Critical,
            Self::GaitEvent { event_type: GaitEventType::StumblePrecursor, .. } => QoSClass::High,
            Self::Alert { priority: AlertPriority::Warning, .. } => QoSClass::High,
            Self::Detection { .. } => QoSClass::Medium,
            Self::NodeSensorData { .. } => QoSClass::Medium,
            Self::NodeHealth { .. } => QoSClass::Low,
            _ => QoSClass::Medium,
        }
    }
}
```

### 12.4 Protocol Timing

```
BLE Connection Interval: 30 ms (default, configurable 7.5-1000 ms)
Message Signing: HMAC-SHA256
Sequence Number: 16-bit rolling
Replay Window: 32 messages

Belt Node Time Master:
  - Broadcasts time sync every 100 ms
  - All nodes compute offset via PTP-style exchange
  
UWB Time Sync (if enabled):
  - Sub-nanosecond precision
  - Exchanges every 1 second
```

---

## 13. Power Management

### 13.1 Power States

| State | CPU | Peripherals | BAN Radio | Wake Source | Power |
|---|---|---|---|---|---|
| **Active** | Full | All on | Connected | N/A | 200-500 mW |
| **Idle** | WFI | Sensors active | Connected | Timer, interrupt | 50-100 mW |
| **Standby** | Off (RAM retained) | Key sensors | Advertising | Radar, timer | 5-20 mW |
| **Deep Sleep** | Off | RTC only | Off | External interrupt | <1 mW |

### 13.2 Belt Node Power States (Linux SoM)

| State | WiFi | Cellular | SLAM | Streaming | Power |
|---|---|---|---|---|---|
| **Minimal** | Idle | Off | Off | Off | ~500 mW |
| **Standard** | Connected | Standby | Off | Off | ~1.4 W |
| **Full Active** | Streaming | Active | On | On | ~4.5 W |
| **Remote Streaming** | Off | Streaming | On | On | ~5.3 W |

### 13.3 Power Profile Configuration

```rust
#[derive(Clone, Debug)]
pub struct PowerProfile {
    pub ble_interval: BleInterval,
    pub sensor_duty_cycle: f32,
    pub camera_power: CameraPowerMode,
    pub uwb_power: UwbPowerMode,
}

#[derive(Clone, Copy, Debug)]
pub enum BleInterval {
    UltraLowLatency,  // 7.5 ms
    LowLatency,       // 15 ms
    Standard,         // 30 ms
    PowerSaving,      // 100 ms
    Sleep,            // 1000 ms
}

impl PowerProfile {
    pub fn ultra_low_latency() -> Self {
        Self {
            ble_interval: BleInterval::UltraLowLatency,
            sensor_duty_cycle: 1.0,
            camera_power: CameraPowerMode::AlwaysOn,
            uwb_power: UwbPowerMode::FullBandwidth,
        }
    }
    
    pub fn standard() -> Self {
        Self {
            ble_interval: BleInterval::Standard,
            sensor_duty_cycle: 0.5,
            camera_power: CameraPowerMode::MotionTriggered,
            uwb_power: UwbPowerMode::TimingOnly,
        }
    }
}
```

---

## 14. Thermal Management

### 14.1 Temperature Monitoring

```rust
pub struct ThermalManager {
    thermistor: NtcThermistor,
    throttling_threshold_c: f32,
    shutdown_threshold_c: f32,
    current_state: ThermalState,
}

impl ThermalManager {
    pub fn check(&mut self) -> ThermalState;
    pub fn get_slam_framerate(&self) -> u8;
}
```

### 14.2 Temperature Thresholds

| Node | Throttling | Shutdown | Notes |
|------|------------|----------|-------|
| Pendant Standard | 42°C | 50°C | Skin contact |
| Pendant 360° | 40°C | 45°C | Higher thermal load |
| Belt MCU | 45°C | 55°C | Enclosure interior |
| Belt Linux SoM | 50°C | 60°C | Active cooling |
| Bracelet | 42°C | 50°C | Skin contact |
| Anklet | 45°C | 55°C | Less sensitivity |

---

## 15. Data Handling — User Configured

### 15.1 Configuration File

```toml
[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
store_360_panorama = true
storage_target = "belt_sd_card"
retention_days = 30
max_storage_mb = 32768
continuous_recording = false
recording_trigger = "on_detection"
include_integrity_chain = true
enable_app_streaming = true
stream_quality = "high"

[pendant_360]
progressive_quality = true
baseline_resolution = "qvga"
refinement_resolution = "720p"
adaptive_bandwidth = true

[extreme_velocity]
enabled = true
mode = "production"
min_velocity_ms = 50
max_velocity_ms = 350
alert_on_detection = true

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

[connectivity.ban]
primary = "ble"
ble_interval = "standard"
uwb_enabled = true
uwb_role = "timing_and_ranging"

[ban.qos]
enabled = true

[ban.antenna_diversity]
enabled = true
antenna_count = 2
switch_threshold_db = 5

[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false

[power]
profile = "standard"
thermal_throttling = true
shutdown_on_overheat = true

[app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9092
enable_remote_access = false
remote_access_token = ""
max_concurrent_streams = 4
```

---

## 16. Build System

```toml
[workspace]
members = ["."]

[package]
name = "sentinel-wear-firmware"
version = "0.1.0"
edition = "2021"

[features]
default = []
belt_node = []
curved_360 = []
event_enhanced = []
uwb = []
hw_camera_switch = []
wifi = ["belt_node"]
cellular = ["belt_node"]
linux_som = ["belt_node"]
slam = ["linux_som"]
full_api = ["linux_som"]
antenna_diversity = []

[[bin]]
name = "pendant_node"
path = "src/bin/pendant_node.rs"

[[bin]]
name = "pendant_360"
path = "src/bin/pendant_360.rs"
required-features = ["curved_360"]

[[bin]]
name = "pendant_event"
path = "src/bin/pendant_event.rs"
required-features = ["event_enhanced"]

[[bin]]
name = "bracelet_left"
path = "src/bin/bracelet_left.rs"

[[bin]]
name = "bracelet_right"
path = "src/bin/bracelet_right.rs"

[[bin]]
name = "belt_node"
path = "src/bin/belt_node.rs"
required-features = ["wifi"]

[[bin]]
name = "belt_node_cellular"
path = "src/bin/belt_node.rs"
required-features = ["wifi", "cellular"]

[[bin]]
name = "belt_node_linux"
path = "src/bin/belt_node_linux.rs"
required-features = ["linux_som", "wifi"]

[[bin]]
name = "belt_node_linux_cellular"
path = "src/bin/belt_node_linux.rs"
required-features = ["linux_som", "wifi", "cellular"]

[[bin]]
name = "anklet_left"
path = "src/bin/anklet_left.rs"

[[bin]]
name = "anklet_right"
path = "src/bin/anklet_right.rs"

[[bin]]
name = "eyewear_node"
path = "src/bin/eyewear_node.rs"

[dependencies]
# Embedded
cortex-m = { version = "0.7", optional = true }
cortex-m-rt = { version = "0.7", optional = true }
embedded-hal = "1.0"
embedded-sdmmc = "0.7"
sha2 = { version = "0.10", default-features = false }
serde = { version = "1.0", default-features = false, features = ["derive"] }
postcard = "1.0"

# Linux
tokio = { version = "1.0", optional = true }
axum = { version = "0.7", optional = true }

[target.'cfg(target_os = "linux")'.dependencies]
libc = "0.2"
```

---

## 17. Legal & Compliance

- **Sensing-only effectors:** No actuation path exists in any firmware binary.
- **Data handling:** User-configured. No firmware-imposed restrictions.
- **WiFi/Cellular:** Belt node firmware only. Radio modules require FCC/CE/ISED certification.
- **SAR compliance:** Critical for wearable devices. Use only body-worn certified BLE modules.
- **Recording laws:** Deployer's responsibility. Firmware does not enforce jurisdiction-specific restrictions.
- **Battery safety:** Hardware protections mandatory. Firmware provides secondary cutoff monitoring.
- **Integrity chain:** SHA-256 hashes generated for all recordings.
- **Export control:** Encryption may be subject to export control regulations.

---

## 18. Debug & Test

### 18.1 Debug Interfaces

- **SWD:** 4-pin debug header (SWCLK, SWDIO, nRST, SWO)
- **RTT:** Non-intrusive logging via Segger RTT
- **UART:** Debug console at 115200 baud, 8N1
- **Test mode:** `std` build for host-side unit testing with mock `embedded-hal`

### 18.2 Test Build

```bash
# Build for hardware
cargo build --target thumbv7em-none-eabihf --bin pendant_node --release
cargo build --target aarch64-unknown-linux-gnu --bin belt_node_linux --release

# Build for testing
cargo test --features test
```

### 18.3 Firmware Verification Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| Power-on self-test | MCU boots, CRC verified | All sensors respond |
| BAN connectivity | Node connects to belt | < 200 ms association |
| Time sync | Node syncs clock with belt | Offset < 1 ms |
| QoS critical | Critical alert transmits immediately | < 5 ms |
| Antenna diversity | RSSI-based switching | Correct antenna selected |
| Recording | Capture and store to SD | File with integrity manifest |
| Streaming | Start/stop stream via BAN | Frames arrive at belt |
| Thermal | Temperature monitoring | Throttling triggers |
| Cellular (belt only) | AT command response | Module responds < 5 s |
| WiFi (belt only) | AP association | Connects < 30 s |

---

## 19. Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2024-01 | Initial specification |
| 0.2 | 2024-01 | Added QoS classes, antenna diversity, RF coexistence, enhanced extreme velocity support |
