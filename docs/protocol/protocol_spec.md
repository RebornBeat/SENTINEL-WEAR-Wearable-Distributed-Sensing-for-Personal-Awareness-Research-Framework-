# Body-Area Mesh Protocol Specification — SENTINEL-WEAR

**Version:** 0.2 | **Status:** Research Reference Design

---

## 1. Scope

This document specifies the protocol used among SENTINEL-WEAR's nodes (pendant, bracelet, belt, anklet, eyewear) and between the belt node and the wearer's companion device (phone, smartwatch, desktop). The protocol is designed for low-latency body-area distributed sensing with deterministic timing guarantees.

The protocol carries:
- Sensor metadata and detection events
- Fusion outputs and track updates
- Alert and notification commands
- Configuration and calibration data
- Health and status telemetry
- Recording availability notifications
- Media stream coordination
- Extreme velocity detection alerts

**What the protocol does NOT carry:**
- Imagery (no camera modalities by default — camera data is handled separately when configured)
- Raw audio waveforms (audio is processed locally; only event classifications and DOA transmitted)
- Commands to active emitters above standard haptic/audio/indicator functions
- Commands to actuators that act on external objects
- Operational-engagement commands of any kind

The protocol's message vocabulary is bounded such that extension to operational active-response use is not possible without redesigning the protocol entirely.

---

## 2. Architectural Constraint: Belt Node as Sole External Gateway

**Critical principle:** Only the belt node connects to external networks. All other nodes communicate exclusively via Body-Area Network (BAN) to the belt node.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR Protocol Layers                            │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN Protocol (BLE 5.x / UWB) ──► Belt Node                   │
│  Eyewear───┘                                       │                         │
│                                                    │                         │
│                              ┌─────────────────────┼─────────────────────┐   │
│                              │                     │                     │   │
│                              ▼                     ▼                     ▼   │
│                         WiFi API             Cellular              BLE     │
│                              │                     │                     │   │
│                              ▼                     ▼                     ▼   │
│                       Companion App         Remote App            Local    │
│                          (Local)            (Remote)              App     │
│                                                                      │      │
│  ⚠️  BELT NODE IS THE ONLY NODE WITH EXTERNAL CONNECTIVITY             │      │
│  ⚠️  ALL OTHER NODES = BAN ONLY                                        │      │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why this matters:**
- Protocol between nodes is strictly BAN (BLE/UWB)
- Companion app protocol runs over WiFi/Cellular/BLE from belt only
- No node other than belt can ever communicate externally
- This is enforced at both hardware and firmware levels

---

## 3. Physical Layer

### 3.1 BAN Physical Layer Options

| Transport | Bandwidth | Power | Latency | Role |
|-----------|-----------|-------|---------|------|
| BLE 5.x | 0.5-2 Mbps | 5-15 mW | 5-30 ms | Primary BAN transport, always present |
| UWB | 4-6 Mbps | 50-150 mW | < 1 ms | Optional: timing, ranging, bandwidth augmentation |

### 3.2 External Communication Physical Layer

| Transport | Bandwidth | Power | Role |
|-----------|-----------|-------|------|
| WiFi 802.11ac/ax | 50-300+ Mbps | 300-500 mW | Primary app connection |
| Cellular LTE/5G | 5-1000+ Mbps | 150-1200 mW | Remote access when WiFi unavailable |
| BLE 5.x | 0.5-2 Mbps | 5-15 mW | Fallback when WiFi unavailable |

### 3.3 Frequency Bands and RF Coexistence

| Radio | Frequency Band | Coexistence Concern |
|-------|---------------|---------------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz and 5 GHz | BLE (2.4 GHz only) |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave radar | 60 GHz | None (separate band) |
| Cellular LTE | 700 MHz - 2.6 GHz | Possible adjacent to 2.4 GHz |
| Cellular 5G Sub-6 | 3.5 GHz | Possible adjacent to UWB |
| Cellular 5G mmWave | 24-47 GHz | None (separate band) |

**Coexistence strategy:**
- WiFi prefers 5 GHz band (eliminates BLE conflict)
- Antenna separation on belt (cellular/WiFi exterior, BLE interior)
- Time-division multiplexing when 2.4 GHz WiFi required

---

## 4. Transport Layer Hierarchy

### 4.1 BLE Transport — Primary BAN

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  BLE 5.x (500 Kbps - 2 Mbps practical)                                       │
│                                                                              │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, IMU orientation, classification results)            │
│  → Low-bandwidth sensor data                                                 │
│  → Alert routing (QoS-classified)                                            │
│  → Always-on, lowest power (5-15 mW)                                         │
│  → Present on ALL nodes                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 UWB Transport — Optional Augmentation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  UWB (4-6 Mbps sustained, 6.8 Mbps peak)                                     │
│                                                                              │
│  → Precision time synchronization (sub-nanosecond)                          │
│  → Precision ranging (cm-level distance between nodes)                       │
│  → Moderate-bandwidth continuous (single camera at 720p-1080p)               │
│  → High-bandwidth BURSTS (compressed clips, short recordings)                │
│  → 360° at 2K-2.5K resolution                                                │
│  → All 8 cameras at QVGA-VGA simultaneously                                  │
│  → Progressive quality strategies (start low-res, improve over time)         │
│  → Optional on nodes, enabled per configuration                              │
│  → Power: 50-150 mW when active                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Bandwidth Capabilities Comparison

| Stream Type | Bandwidth Required | BLE | UWB | WiFi |
|-------------|-------------------|-----|-----|------|
| Detection metadata | < 10 Kbps | ✅ | ✅ | ✅ |
| IMU stream (200 Hz batched) | ~50 Kbps | ✅ | ✅ | ✅ |
| Gait event | < 5 Kbps | ✅ | ✅ | ✅ |
| 8 cameras × QVGA MJPEG | 0.5-1 Mbps | ❌ | ✅ | ✅ |
| 8 cameras × VGA H.264 | 2-4 Mbps | ❌ | ✅ | ✅ |
| Stitched 360° at 2K H.264 | 3-4 Mbps | ❌ | ✅ | ✅ |
| Stitched 360° at 4K H.264 | 8-15 Mbps | ❌ | ❌ | ✅ |
| Single camera 720p-1080p | 1-3 Mbps | ❌ | ✅ | ✅ |

### 4.4 Tiered Progressive Quality Over UWB

When bandwidth is constrained, UWB supports progressive quality improvement:

```
Phase 1 (Instant):     8 cameras at QVGA → complete 360° view within 1 second
                       Total bandwidth: ~500 Kbps
                       
Phase 2 (Progressive): Send higher-res keyframes sequentially over 5-10 seconds
                       Camera 0: 720p keyframe → belt updates stitching
                       Camera 1: 720p keyframe → belt updates stitching
                       ...
                       Combined with baseline: ~2-4 Mbps total
                       
Phase 3 (Continuous):  Maintain QVGA baseline continuously
                       Periodically send higher-res keyframes
                       Final panorama quality improves over time
```

---

## 5. BLE Scheduling and Determinism

### 5.1 Time-Slotted Scheduling

**Problem:** If all nodes transmit simultaneously over BLE, collisions and latency spikes occur.

**Solution:** Deterministic time-slotted scheduling with the belt node as coordinator.

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
├── Slot 25-28 ms: RESERVED (critical alerts, extreme velocity detection)
└── Slot 28-30 ms: Reserved (retransmits, eyewear)
```

**Guarantee:** Every node gets its slot every cycle. Latency is bounded.

### 5.2 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture, research |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### 5.3 Node Priority for Bandwidth Allocation

When multiple nodes have data to transmit, higher-priority nodes are served first:

| Priority | Node | Reason |
|----------|------|--------|
| 1 (Highest) | Belt IMU | Torso reference, all body-frame fusion depends on it |
| 2 | Anklets | Gait phase timing, critical for stumble detection |
| 3 | Pendant | Primary sensing, highest information value |
| 4 | Bracelets | Secondary sensing |
| 5 (Lowest) | Eyewear | Optional, supplementary |

**Override:** Critical alerts (fall detected, extreme velocity detection) bypass normal scheduling.

### 5.4 Dynamic Scheduling

Context-aware scheduling adjusts priorities based on current state:

| Context | Schedule Adjustment |
|---------|---------------------|
| Walking detected | Prioritize anklets (gait timing critical) |
| Threat detected | Prioritize pendant (sensing critical) |
| Idle state | Extend intervals (power saving) |
| Streaming active | Shift heavy traffic to UWB |
| Extreme velocity alert | Preempt all other traffic for detection message |

---

## 6. QoS Classes

### 6.1 Traffic Classification

| QoS Class | Priority | Traffic Types | Latency Guarantee |
|-----------|----------|---------------|-------------------|
| Critical | Highest | Extreme velocity detection, fall detection, emergency alerts | < 5 ms |
| High | Second | Gait anomaly, stumble precursor, threat detection | < 20 ms |
| Medium | Third | Presence detection, position tracking, sensor updates | < 50 ms |
| Low | Fourth | Battery status, health telemetry, non-critical diagnostics | < 500 ms |

### 6.2 Critical QoS Behavior

```rust
/// Messages with QoS = Critical preempt all other traffic.
/// They are transmitted immediately, regardless of scheduled slot.
pub enum CriticalMessage {
    ExtremeVelocityDetection(ExtremeVelocityEvent),
    FallDetected(FallEvent),
    EmergencyAlert(EmergencyAlert),
}

impl CriticalMessage {
    /// Critical messages bypass the normal scheduler queue.
    /// They are transmitted at the next available microslot (< 1 ms).
    pub fn transmit_immediately(&self, radio: &mut BLEController) {
        radio.preempt_and_transmit(self);
    }
}
```

### 6.3 Reserved Slot for Critical Messages

Every BLE connection event reserves a dedicated slot for critical messages:

```
Slot 25-28 ms: RESERVED for critical alerts
```

This ensures that even under high load, critical messages have a guaranteed transmission opportunity within each 30 ms cycle.

---

## 7. Antenna Diversity

### 7.1 Purpose

The human body is a significant RF absorber at 2.4 GHz:
- Torso absorption: 10-30 dB attenuation
- Arm position changes: 5-15 dB variation
- Orientation changes: 10-20 dB variation

Antenna diversity mitigates body shadowing without adding RF interference.

### 7.2 Implementation

```
Belt Node BLE Radio
    │
    ├── Antenna Switch
    │       ├── Left-side antenna (toward wearer's left)
    │       └── Right-side antenna (toward wearer's right)
    │
    └── Radio selects best antenna dynamically
```

**Key properties:**
- ONE transmitter active at any time
- ONE receiver active at any time
- NO self-interference (unlike multi-radio)
- Antenna selection time: < 1 microsecond

### 7.3 Antenna Selection Algorithm

```rust
/// RSSI-based antenna diversity selection
pub fn select_best_antenna(&mut self, packet_stats: &PacketStats) -> Antenna {
    let left_rssi = self.read_rssi(Antenna::Left);
    let right_rssi = self.read_rssi(Antenna::Right);
    
    // Hysteresis: don't switch too frequently
    if left_rssi > right_rssi + HYSTERESIS_DB {
        Antenna::Left
    } else if right_rssi > left_rssi + HYSTERESIS_DB {
        Antenna::Right
    } else {
        self.current_antenna // Stay on current
    }
}
```

---

## 8. UWB Role Configuration

### 8.1 UWB Roles

| Role | Capabilities | Power | Use Case |
|------|-------------|-------|----------|
| `timing_only` | Sub-ns time sync, cm-level ranging | ~50 mW | Precision body-frame reconstruction |
| `timing_and_ranging` | All timing + distance measurement + occasional bursts | ~80 mW | Enhanced body-frame accuracy |
| `full_bandwidth` | All timing + sustained 5-6 Mbps streaming | 100-150 mW | Camera streaming over UWB |

### 8.2 UWB Message Types

UWB carries different message types than BLE:

| Message Type | BLE | UWB | Reason |
|--------------|-----|-----|--------|
| Control/commands | ✅ | ❌ | Low bandwidth, BLE sufficient |
| Detection metadata | ✅ | ❌ | Low bandwidth, BLE sufficient |
| Health/telemetry | ✅ | ❌ | Low bandwidth, BLE sufficient |
| Camera burst | ❌ | ✅ | High bandwidth |
| Radar burst | ❌ | ✅ | High bandwidth |
| Audio clip | ❌ | ✅ | Moderate bandwidth |
| SLAM data | ❌ | ✅ | High bandwidth |

---

## 9. Message Types — Complete Specification

### 9.1 Sensor Metadata Messages

#### NodeSensorData

```rust
/// Core sensor data from any node
pub struct NodeSensorData {
    /// Timestamp in milliseconds since boot (node-local clock)
    pub timestamp_ms: u64,
    
    /// Radar-derived presence/motion data
    pub radar_metadata: RadarMetadata,
    
    /// IMU orientation as quaternion [w, x, y, z]
    pub imu_quaternion: [f32; 4],
    
    /// Linear acceleration in node frame
    pub linear_accel: [f32; 3],
    
    /// Source node identifier
    pub node_id: NodeId,
    
    /// Raw audio clip available on local storage (if configured)
    pub audio_clip_available: bool,
    
    /// Path to raw audio clip on node storage (if stored)
    pub audio_clip_path: Option<String>,
}

pub struct RadarMetadata {
    /// Detection confidence (0.0 - 1.0)
    pub confidence: f32,
    
    /// Radial velocity in m/s (from Doppler)
    pub radial_velocity_ms: f32,
    
    /// Range to detected object in meters
    pub range_m: f32,
    
    /// Azimuth angle in radians (node frame)
    pub azimuth_rad: f32,
    
    /// Elevation angle in radians (node frame)
    pub elevation_rad: f32,
    
    /// Activity classification (if detected)
    pub activity_class: Option<ActivityClass>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ActivityClass {
    Stationary,
    Walking,
    Running,
    Falling,
    Unknown,
}
```

#### GaitEvent

```rust
/// Gait phase event from anklet nodes
pub struct GaitEvent {
    /// Source node (left or right anklet)
    pub node_id: NodeId,
    
    /// Timestamp
    pub timestamp_ms: u64,
    
    /// Current gait phase
    pub phase: GaitPhase,
    
    /// Phase transition confidence
    pub confidence: f32,
    
    /// Heel-strike impact magnitude (g-forces)
    pub impact_g: Option<f32>,
    
    /// Stride duration in milliseconds
    pub stride_duration_ms: Option<u16>,
    
    /// Step regularity coefficient (0.0 - 1.0)
    pub regularity: Option<f32>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum GaitPhase {
    Stance,
    Swing,
    HeelStrike,
    ToeOff,
    MidStance,
}
```

#### AcousticEvent

```rust
/// Acoustic event detection from microphone array
pub struct AcousticEvent {
    /// Source node
    pub node_id: NodeId,
    
    /// Timestamp
    pub timestamp_ms: u64,
    
    /// Event classification
    pub event_class: String,
    
    /// Direction of arrival (azimuth, elevation) in radians
    pub direction_of_arrival: (f32, f32),
    
    /// Classification confidence
    pub confidence: f32,
    
    /// Path to raw audio clip if stored locally
    pub audio_clip_path: Option<String>,
}
```

### 9.2 Fusion Output Messages

#### TrackedEntity

```rust
/// Fused track from belt controller
pub struct TrackedEntity {
    /// Unique track identifier
    pub id: String,
    
    /// Position in body frame (meters)
    pub position: [f32; 3],
    
    /// Velocity in body frame (m/s)
    pub velocity: [f32; 3],
    
    /// Classification hint
    pub classification_hint: ClassificationHint,
    
    /// Detection confidence
    pub confidence: f32,
    
    /// PentaTrack prediction centers (future positions)
    pub prediction_centers: Vec<[f32; 3]>,
    
    /// Anomaly flags
    pub anomaly_flags: Vec<AnomalyFlag>,
    
    /// Source detections that contributed to this track
    pub source_nodes: Vec<NodeId>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ClassificationHint {
    Human,
    Pet,
    Vehicle,
    Object,
    Unknown,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AnomalyFlag {
    UnusualVelocity,
    ErraticMovement,
    Approaching,
    Retreating,
    Loitering,
    PotentialThreat,
}
```

### 9.3 Alert Messages

#### AlertEvent

```rust
/// Alert event to be routed to haptic/audio/app
pub struct AlertEvent {
    /// Alert identifier
    pub id: String,
    
    /// Alert class
    pub alert_class: AlertClass,
    
    /// Urgency level
    pub urgency: Urgency,
    
    /// Source track/entity ID (if applicable)
    pub source_track: Option<String>,
    
    /// Target nodes for haptic output
    pub haptic_targets: Vec<NodeId>,
    
    /// Haptic pattern
    pub haptic_pattern: HapticPattern,
    
    /// Timestamp
    pub timestamp_ms: u64,
    
    /// Additional context
    pub context: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AlertClass {
    HumanApproaching,
    VehicleNear,
    GaitAnomaly,
    FallDetected,
    FastObjectDetected,
    ZoneViolation,
    UnknownThreat,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Urgency {
    Info,
    Warning,
    Critical,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum HapticPattern {
    SinglePulse { duration_ms: u16 },
    DoublePulse { duration_ms: u16, gap_ms: u16 },
    TriplePulse { duration_ms: u16, gap_ms: u16 },
    SustainedBuzz { duration_ms: u16 },
    RapidPulses { count: u8, duration_ms: u16, gap_ms: u16 },
    Custom { pattern: Vec<u16> },
}
```

#### ExtremeVelocityEvent

```rust
/// Critical: Fast-moving object detected
/// This message has QoS = Critical and preempts all other traffic
pub struct ExtremeVelocityEvent {
    /// Detection timestamp (microsecond precision)
    pub timestamp_us: u64,
    
    /// Source node (pendant or eyewear with event camera)
    pub source_node: NodeId,
    
    /// Detected velocity in m/s
    pub velocity_ms: f32,
    
    /// Direction of approach in body frame (radians)
    pub direction_rad: f32,
    
    /// Estimated distance at detection
    pub distance_m: f32,
    
    /// Detection confidence
    pub confidence: f32,
    
    /// Doppler radar signature path (if stored)
    pub doppler_signature_path: Option<String>,
    
    /// Event camera frame path (if stored)
    pub event_frame_path: Option<String>,
}
```

### 9.4 Recording and Media Messages

#### RecordingAvailable

```rust
/// Notification that a recording is available for retrieval
pub struct RecordingAvailable {
    /// Source node
    pub node_id: NodeId,
    
    /// Path to recording on node storage
    pub recording_path: String,
    
    /// Recording start timestamp
    pub start_timestamp_ms: u64,
    
    /// Duration in milliseconds
    pub duration_ms: u32,
    
    /// File size in bytes
    pub size_bytes: u64,
    
    /// Trigger that started recording
    pub trigger: RecordingTrigger,
    
    /// Format
    pub format: String,
    
    /// Modalities included
    pub modalities: Vec<Modality>,
    
    /// SHA-256 hash of recording file
    pub sha256: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RecordingTrigger {
    OnDetection,
    OnAlert,
    Manual,
    Continuous,
    Scheduled,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Modality {
    Video,
    Audio,
    RadarIQ,
    LidarPointCloud,
    IMU,
    Event,
}
```

#### MediaStreamAvailable

```rust
/// Live media stream is available from this node
pub struct MediaStreamAvailable {
    /// Source node
    pub node_id: NodeId,
    
    /// Stream endpoint port
    pub stream_port: u16,
    
    /// Stream format
    pub format: StreamFormat,
    
    /// Stream quality
    pub quality: StreamQuality,
    
    /// Timestamp stream started
    pub timestamp_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum StreamFormat {
    H264,
    H265,
    MJPEG,
    RawRGB,
    PCM,
    AAC,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum StreamQuality {
    Low,
    Medium,
    High,
    Raw,
}
```

#### MediaStreamEnded

```rust
/// Live media stream has ended
pub struct MediaStreamEnded {
    /// Source node
    pub node_id: NodeId,
    
    /// Timestamp stream ended
    pub timestamp_ms: u64,
    
    /// Reason for ending
    pub reason: StreamEndReason,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum StreamEndReason {
    UserStopped,
    Timeout,
    Error,
    NodeDisconnected,
}
```

#### PanoramaFrameAvailable

```rust
/// 360° panorama frame available from curved pendant
pub struct PanoramaFrameAvailable {
    /// Source node
    pub node_id: NodeId,
    
    /// Frame timestamp
    pub timestamp_ms: u64,
    
    /// Panorama width in pixels
    pub width: u32,
    
    /// Panorama height in pixels
    pub height: u32,
    
    /// Path to frame on node storage (if stored)
    pub frame_path: Option<String>,
    
    /// Quality level of this frame
    pub quality: StreamQuality,
}
```

### 9.5 Configuration Messages

#### ConfigurationUpdate

```rust
/// Configuration update from belt to node
pub struct ConfigurationUpdate {
    /// Target node (or broadcast to all)
    pub target: Option<NodeId>,
    
    /// Configuration section being updated
    pub section: ConfigSection,
    
    /// New value (TOML fragment)
    pub value: String,
    
    /// Sequence number for ordering
    pub sequence: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ConfigSection {
    Sensors,
    DataHandling,
    Privacy,
    Alerts,
    Power,
    Connectivity,
}
```

#### CalibrationCommand

```rust
/// Calibration command from belt to node(s)
pub struct CalibrationCommand {
    /// Calibration phase
    pub phase: CalibrationPhase,
    
    /// Target nodes (empty = all)
    pub targets: Vec<NodeId>,
    
    /// Timestamp command issued
    pub timestamp_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum CalibrationPhase {
    StartNeutralPose,
    CompleteNeutralPose,
    StartWalkThrough,
    CompleteWalkThrough,
    StartCameraCalibration,
    CompleteCameraCalibration,
    Cancel,
}
```

### 9.6 Health and Status Messages

#### NodeHealth

```rust
/// Periodic health report from node
pub struct NodeHealth {
    /// Source node
    pub node_id: NodeId,
    
    /// Battery percentage (0-100)
    pub battery_percent: u8,
    
    /// Battery voltage in millivolts
    pub battery_mv: u16,
    
    /// Charging status
    pub charging: bool,
    
    /// Sensor health status
    pub sensor_status: Vec<SensorStatus>,
    
    /// BLE link quality
    pub ble_rssi_dbm: i8,
    
    /// UWB link quality (if applicable)
    pub uwb_rssi_dbm: Option<i8>,
    
    /// Timestamp
    pub timestamp_ms: u64,
    
    /// Uptime in seconds
    pub uptime_s: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SensorStatus {
    pub sensor_type: String,
    pub status: SensorHealth,
    pub error_code: Option<u16>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum SensorHealth {
    Healthy,
    Degraded,
    Failed,
}
```

#### TimeSyncExchange

```rust
/// Precision time synchronization exchange (PTP-style)
pub struct TimeSyncExchange {
    /// t1: Time request sent (master clock)
    pub t1_ns: u64,
    
    /// t2: Time request received (slave clock)
    pub t2_ns: u64,
    
    /// t3: Time response sent (slave clock)
    pub t3_ns: u64,
    
    /// t4: Time response received (master clock)
    pub t4_ns: u64,
}
```

### 9.7 Control Messages

#### HapticCommand

```rust
/// Haptic actuator command from belt to node
pub struct HapticCommand {
    /// Target node(s)
    pub targets: Vec<NodeId>,
    
    /// Haptic pattern
    pub pattern: HapticPattern,
    
    /// Intensity (0-100)
    pub intensity: u8,
    
    /// Optional: repeat count
    pub repeat: Option<u8>,
}
```

#### StartStreamToBelt

```rust
/// Command to node to start streaming to belt
pub struct StartStreamToBelt {
    /// Target node
    pub node_id: NodeId,
    
    /// Stream format
    pub format: StreamFormat,
    
    /// Stream quality
    pub quality: StreamQuality,
    
    /// Duration limit in seconds (0 = unlimited)
    pub max_duration_s: u16,
}
```

#### StopStreamToBelt

```rust
/// Command to node to stop streaming
pub struct StopStreamToBelt {
    /// Target node
    pub node_id: NodeId,
}
```

---

## 10. Complete Message Enum

```rust
/// All BAN protocol messages
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum SentinelBanMessage {
    // === Sensor Data ===
    NodeSensorData(NodeSensorData),
    GaitEvent(GaitEvent),
    AcousticEvent(AcousticEvent),
    
    // === Fusion Output ===
    TrackedEntity(TrackedEntity),
    
    // === Alerts ===
    AlertEvent(AlertEvent),
    ExtremeVelocityEvent(ExtremeVelocityEvent),
    
    // === Recording and Media ===
    RecordingAvailable(RecordingAvailable),
    MediaStreamAvailable(MediaStreamAvailable),
    MediaStreamEnded(MediaStreamEnded),
    PanoramaFrameAvailable(PanoramaFrameAvailable),
    
    // === Configuration ===
    ConfigurationUpdate(ConfigurationUpdate),
    CalibrationCommand(CalibrationCommand),
    
    // === Health and Status ===
    NodeHealth(NodeHealth),
    TimeSyncExchange(TimeSyncExchange),
    
    // === Control ===
    HapticCommand(HapticCommand),
    StartStreamToBelt(StartStreamToBelt),
    StopStreamToBelt(StopStreamToBelt),
    
    // === System ===
    Heartbeat {
        node_id: NodeId,
        timestamp_ms: u64,
        sequence: u64,
    },
    
    Ack {
        message_id: String,
        status: AckStatus,
        timestamp_ms: u64,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AckStatus {
    Ok,
    Error { code: u16, message: String },
}
```

---

## 11. Message Signing and Security

### 11.1 HMAC-SHA256 Signing

All BAN messages are signed with HMAC-SHA256:

```rust
pub struct SignedMessage {
    /// Message payload
    pub payload: SentinelBanMessage,
    
    /// Sequence number (replay protection)
    pub sequence: u64,
    
    /// Timestamp
    pub timestamp_ms: u64,
    
    /// HMAC-SHA256 signature
    pub signature: [u8; 32],
    
    /// Source node ID
    pub source: NodeId,
}

impl SignedMessage {
    pub fn new(payload: SentinelBanMessage, key: &[u8], sequence: u64) -> Self {
        let timestamp_ms = current_time_ms();
        let source = payload.source_node();
        
        // Create message to sign
        let message = format!("{}:{}:{}:{}", 
            sequence, timestamp_ms, source, serde_json::to_string(&payload).unwrap()
        );
        
        // Compute HMAC
        let mut mac = HmacSha256::new_from_slice(key).unwrap();
        mac.update(message.as_bytes());
        let signature = mac.finalize().into_bytes();
        
        Self {
            payload,
            sequence,
            timestamp_ms,
            signature: signature.into(),
            source,
        }
    }
    
    pub fn verify(&self, key: &[u8], expected_sequence: u64) -> Result<(), ProtocolError> {
        // Check sequence number (replay protection)
        if self.sequence <= expected_sequence {
            return Err(ProtocolError::ReplayAttack);
        }
        
        // Recompute and verify signature
        let message = format!("{}:{}:{}:{}", 
            self.sequence, self.timestamp_ms, self.source, 
            serde_json::to_string(&self.payload).unwrap()
        );
        
        let mut mac = HmacSha256::new_from_slice(key).unwrap();
        mac.update(message.as_bytes());
        let expected = mac.finalize().into_bytes();
        
        if self.signature != expected {
            return Err(ProtocolError::InvalidSignature);
        }
        
        Ok(())
    }
}
```

### 11.2 Key Establishment

Keys are established during pairing:

```
1. User initiates pairing via companion app
2. Belt generates random 256-bit key
3. Key transferred to node via secure BLE pairing (LE Secure Connections)
4. Key stored in secure element or encrypted flash on both ends
5. All subsequent messages use this key for signing
```

### 11.3 Replay Protection

- Each message includes a monotonically increasing sequence number
- Belt tracks expected sequence per node
- Messages with sequence ≤ expected are rejected as replay attacks
- Sequence numbers wrap at u64::MAX (practically impossible to exhaust)

---

## 12. Wire Format

### 12.1 Binary Encoding

Messages are encoded using a compact binary format:

```
┌─────────────────────────────────────────────────────────────────┐
│ Message Frame                                                   │
├─────────────────────────────────────────────────────────────────┤
│ [0-1]   Start marker (0xABCD)                                   │
│ [2-3]   Payload length (little-endian u16)                      │
│ [4-5]   Message type ID (u16 enum)                              │
│ [6-9]   Sequence number (little-endian u32)                     │
│ [10-13] Timestamp (little-endian u32, ms since boot)            │
│ [14-N]  Payload (postcard-encoded)                              │
│ [N+1-N+32] HMAC-SHA256 signature                                │
│ [N+33-N+34] End marker (0xDCBA)                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 JSON Encoding (Debug/API)

For debugging and API access, messages can also be JSON-encoded:

```json
{
  "type": "NodeSensorData",
  "timestamp_ms": 1710509422004,
  "radar_metadata": {
    "confidence": 0.89,
    "radial_velocity_ms": 1.2,
    "range_m": 4.5,
    "azimuth_rad": 0.78,
    "elevation_rad": -0.15,
    "activity_class": "Walking"
  },
  "imu_quaternion": [0.98, 0.02, -0.03, 0.17],
  "linear_accel": [0.1, -0.05, 9.8],
  "node_id": "pendant",
  "audio_clip_available": false
}
```

---

## 13. Protocol Flow Examples

### 13.1 Normal Operation Flow

```
1. Belt broadcasts heartbeat every 100 ms
2. Each node transmits sensor data in its scheduled slot
3. Belt fuses data, produces TrackedEntity messages
4. Alerts are generated based on track classification
5. HapticCommand sent to appropriate node(s)
6. Companion app receives updates via WiFi from belt
```

### 13.2 Extreme Velocity Detection Flow

```
1. Doppler radar detects high-velocity object (> 50 m/s)
2. Event camera confirms trajectory (< 1 ms)
3. ExtremeVelocityEvent created with QoS = Critical
4. Message transmitted immediately, preempting normal scheduling
5. Belt receives within < 5 ms
6. Alert generated and routed to haptic nodes
7. User receives haptic alert within < 10 ms total latency
8. Event data optionally stored for forensic analysis
```

### 13.3 360° Streaming Flow

```
1. Companion app requests live 360° view
2. Belt sends StartStreamToBelt to 360° pendant
3. Pendant begins progressive quality transmission:
   a. QVGA baseline (all 8 cameras) via UWB immediately
   b. Higher-res keyframes sequentially over 5-10 seconds
   c. Continuous baseline maintained
4. Belt stitches panorama (if pendant-side stitching not used)
5. Belt streams to companion app via WiFi
6. App displays live 360° view with quality improving over time
7. User ends stream; StopStreamToBelt sent
8. MediaStreamEnded confirmation
```

---

## 14. Configuration Reference

```toml
[ban]
primary = "ble"                       # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

[ban.scheduling]
mode = "dynamic"                      # "static" | "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
extreme_velocity_qos = "critical"
extreme_velocity_preempt = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28

[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5

[ban.qos.high]
max_latency_ms = 20

[ban.qos.medium]
max_latency_ms = 50

[ban.qos.low]
max_latency_ms = 500

[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
hysteresis_db = 2

[ban.uwb]
enabled = false
role = "timing_and_ranging"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[ban.security]
message_signing = true
key_rotation_days = 30
```

---

## 15. Versioning

The protocol is versioned with the repository:

```
Protocol Version: MAJOR.MINOR
- MAJOR: Breaking changes (message format, security model)
- MINOR: New message types, extended fields (backward compatible)
```

Changes require:
- Maintainer review
- CHANGELOG entry tagged `[protocol]`
- Version field in message header

**Current Version:** 0.2

---

## 16. What the Protocol Does NOT Carry

| Excluded Content | Reason |
|-----------------|--------|
| Imagery | Privacy, bandwidth; camera data handled separately |
| Raw audio waveforms | Privacy; processed locally, only events transmitted |
| Commands to active emitters | None present in hardware |
| Commands to actuators acting on external objects | Explicitly out of scope |
| Operational-engagement commands | Explicitly out of scope |

The protocol vocabulary is bounded to prevent extension to active-response operations without complete redesign.

---

**End of Body-Area Mesh Protocol Specification**
