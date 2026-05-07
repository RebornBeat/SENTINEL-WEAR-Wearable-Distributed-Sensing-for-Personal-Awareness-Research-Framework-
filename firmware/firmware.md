# Firmware Architecture — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V), Linux (Belt Node SoM)

---

## 1. Purpose

This document specifies the firmware architecture for SENTINEL-WEAR's sensing nodes. The firmware runs on resource-constrained MCUs embedded in jewelry-form-factor wearables (pendant, bracelets, anklets, eyewear) and on Linux-class SoMs in the belt node.

The firmware is implemented in Rust using the `no_std` ecosystem for MCUs and standard Rust for Linux SoM belt nodes. It is responsible for:
- Driving sensors (mmWave radar, IMU, LiDAR/ToF, acoustic, environmental, event, cameras).
- Performing node-local processing to reduce BAN bandwidth.
- Power management (critical for 24–72 hour battery life).
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
#[cfg(not(feature = "belt_node"))]
compile_error!("WiFi and cellular features are belt-node-only");

// Pendant node binary
#[cfg(all(feature = "wifi", not(feature = "belt_node")))]
compile_error!("WiFi is not available on non-belt nodes");
```

### Belt Node Firmware Capabilities

The belt node firmware is the only firmware variant that links against:

| Driver | Belt Only? | Purpose |
|--------|------------|---------|
| `wifi.rs` | ✅ Yes | Connect to home network, serve companion app API |
| `cellular.rs` | ✅ Yes | LTE/5G connectivity for remote access |
| `ban_hub.rs` | ✅ Yes (as master) | Coordinate all BAN communication |
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
├── Slot 25-30 ms: Reserved (alerts, retransmits, eyewear)
```

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

### Firmware Configuration

```rust
/// BLE scheduling configuration
#[derive(Clone, Copy, Debug)]
pub struct BleScheduleConfig {
    /// Connection interval in milliseconds
    pub connection_interval_ms: u16,
    /// Node priority ordering
    pub node_priority: [NodeId; 6],
    /// Slot allocation per node in microseconds
    pub slot_allocation_us: [(NodeId, u16); 6],
    /// Reserved slot for alerts (microseconds)
    pub alert_slot_us: u16,
}

impl Default for BleScheduleConfig {
    fn default() -> Self {
        Self {
            connection_interval_ms: 30,
            node_priority: [
                NodeId::Belt,      // Priority 1
                NodeId::AnkletL,   // Priority 2
                NodeId::AnkletR,   // Priority 2
                NodeId::Pendant,   // Priority 3
                NodeId::BraceletL, // Priority 4
                NodeId::BraceletR, // Priority 4
            ],
            slot_allocation_us: [
                (NodeId::Pendant, 3000),
                (NodeId::AnkletL, 3000),
                (NodeId::AnkletR, 3000),
                (NodeId::BraceletL, 3000),
                (NodeId::BraceletR, 3000),
                (NodeId::Eyewear, 2000),
            ],
            alert_slot_us: 5000,
        }
    }
}
```

---

## 5. Target Platforms

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

## 6. Workspace Structure

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
    │   ├── camera.rs               # Single camera (pendant, bracelet, anklet)
    │   ├── camera_360.rs           # 360° curved pendant multi-camera
    │   ├── storage.rs              # SD card + integrity chain
    │   ├── wifi.rs                 # Belt node only
    │   ├── cellular.rs             # Belt node only
    │   └── uwb.rs                  # Optional on all nodes
    ├── logic/
    │   ├── mod.rs
    │   ├── body_frame_local.rs
    │   ├── presence_detection.rs
    │   ├── gait_analysis.rs        # On-node gait event detection
    │   ├── extreme_velocity.rs     # Doppler + event fusion for fast objects
    │   ├── recording_manager.rs
    │   ├── power_management.rs
    │   ├── thermal_management.rs   # Temperature monitoring and throttling
    │   └── progressive_quality.rs  # Tiered quality for bandwidth adaptation
    └── ban_radio/
        ├── mod.rs
        ├── ble_scheduler.rs        # Time-slotted BLE scheduling
        ├── uwb_driver.rs           # UWB timing/ranging/bandwidth
        └── protocol.rs             # BAN message definitions
```

---

## 7. Node Binary Descriptions

### 7.1 Belt Node — MCU Variant (`belt_node.rs`)

**Role:** Primary compute, BAN hub, **sole external network gateway**, API server.

**Drivers used:** `mmwave.rs`, `imu.rs`, `environmental.rs`, `storage.rs`, `wifi.rs`, `cellular.rs`

**Firmware responsibilities:**
- **BAN hub:** Coordinates all inter-node communication, time-sync master, BLE scheduler
- **WiFi connectivity:** Connects to home network; serves minimal API server
- **Cellular SIM:** Maintains LTE connection when configured; forwards alerts
- **Recording manager:** Aggregates `RecordingAvailable` events from all nodes; manages belt SD storage
- **Sensor fusion coordination:** Routes all node data to embedded fusion algorithms

**Limitations (MCU variant):**
- No dense SLAM (insufficient compute)
- Limited camera stream handling (1-2 streams)
- Minimal API server (REST only, no WebSocket)

### 7.2 Belt Node — Linux SoM Variant (`belt_node_linux.rs`)

**Role:** Full capability belt node with SLAM, 360° processing, complete API server.

**Additional drivers:** `slam.rs`, `stitching.rs`, `api_server.rs`

**Additional responsibilities:**
- **SLAM processing:** Dense world model construction
- **360° stitching:** Real-time panorama from pendant camera array
- **Full API server:** REST + WebSocket + RTSP
- **Progressive quality:** Receives tiered streams from 360° pendant, reconstructs high quality
- **Thermal management:** Monitors enclosure temperature, throttles SLAM if needed

### 7.3 Pendant Node — Standard/Medallion (`pendant_node.rs`)

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

### 7.4 Pendant Node — 360° Curved (`pendant_360.rs`)

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
    cameras: [CameraDriver; 8],
    fsync_gpio: GpioPin<Output<PushPull>>,
    calibration: CalibrationData,
}

impl Camera360Capture {
    /// Capture all cameras simultaneously
    pub fn capture_sync(&mut self) -> Result<[Frame; 8], CaptureError> {
        // Assert FSYNC - all cameras start exposure simultaneously
        self.fsync_gpio.set_high();
        
        // Wait for exposure time
        self.delay.delay_us(self.exposure_us);
        
        // Deassert FSYNC
        self.fsync_gpio.set_low();
        
        // Read frames from all cameras
        let mut frames = [Frame::empty(); 8];
        for (i, camera) in self.cameras.iter_mut().enumerate() {
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
pub enum ProgressiveQualityPhase {
    /// Instant baseline: all cameras at QVGA
    Baseline,
    /// Progressive refinement: sequential keyframes at higher res
    Refinement,
    /// Continuous improvement
    Continuous,
}

impl Camera360Capture {
    /// Encode with tiered quality
    pub fn encode_progressive(
        &mut self,
        frames: [Frame; 8],
        phase: ProgressiveQualityPhase,
        available_bandwidth_bps: u32,
    ) -> Result<Vec<u8>, EncodeError> {
        match phase {
            ProgressiveQualityPhase::Baseline => {
                // All cameras at QVGA, MJPEG
                self.encode_all_at_resolution(Resolution::QVGA, Codec::MJPEG)
            }
            ProgressiveQualityPhase::Refinement => {
                // Select subset of cameras for high-res keyframes
                // Based on available bandwidth
                let cameras_to_refine = (available_bandwidth_bps / 500_000) as usize;
                self.encode_keyframes(cameras_to_refine.min(8), Resolution::P720, Codec::H264)
            }
            ProgressiveQualityPhase::Continuous => {
                // Maintain baseline + periodic refinements
                self.encode_continuous()
            }
        }
    }
}
```

### 7.5 Pendant Node — Event-Enhanced (Variant E)

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
                        // Alert belt immediately
                        ban.send_alert(AlertClass::FastObjectDetected {
                            velocity_ms: detection.velocity_ms,
                            bearing_rad: detection.bearing,
                            distance_m: detection.range,
                        });
                        
                        // Capture high-res from conventional cameras
                        let frames = conventional_cams.capture_sync();
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

### 7.6 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

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
    // Which bracelet should fire?
    let left_bearing = if approach_bearing_rad > std::f32::consts::PI {
        (2.0 * std::f32::consts::PI) - approach_bearing_rad
    } else {
        approach_bearing_rad
    };
    
    let right_bearing = approach_bearing_rad;
    
    // Left bracelet fires for left-side approaches (bearing ~PI to 2PI)
    // Right bracelet fires for right-side approaches (bearing ~0 to PI)
    if left_bearing > std::f32::consts::FRAC_PI_2 && left_bearing < 1.5 * std::f32::consts::PI {
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

### 7.7 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

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
    stride_samples: CircularBuffer<[ImuSample; 200]>,
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
                impact_g: (accel_magnitude * 10.0) as u8, // Convert to 0-255 scale
                timestamp_ms: sample.timestamp_ms,
            });
        }
        
        None
    }
    
    /// Detect pre-stumble signature
    pub fn detect_stumble_risk(&self) -> Option<StumblePrecursor> {
        // Analyze recent stride patterns
        // Look for: irregular cadence, asymmetry, reduced clearance
        // ...
    }
}
```

### 7.8 Eyewear Node (`eyewear_node.rs`)

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
    }
}
```

---

## 8. Driver Layer

### 8.1 `mmwave.rs` — Standard mmWave Radar

```rust
/// mmWave radar driver for presence detection
pub struct MmWaveDriver<SPI: SpiDevice> {
    spi: SPI,
    config: MmWaveConfig,
}

impl<SPI: SpiDevice> MmWaveDriver<SPI> {
    /// Initialize radar
    pub fn init(&mut self) -> Result<(), RadarError>;
    
    /// Perform single scan
    pub fn scan(&mut self) -> Result<RadarScan, RadarError>;
    
    /// Get micro-Doppler activity signature
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

### 8.2 `mmwave_doppler.rs` — Extended Doppler for Extreme Velocity

```rust
/// Doppler radar driver configured for extreme velocity detection
pub struct DopplerRadarDriver<SPI: SpiDevice> {
    spi: SPI,
    config: DopplerConfig,
    /// Extended velocity range: can detect up to max_velocity_ms
    max_velocity_ms: f32,
}

/// Doppler configuration for projectile detection
#[derive(Clone, Copy, Debug)]
pub struct DopplerConfig {
    /// Maximum velocity to detect (m/s)
    /// Standard automotive: ~18 m/s
    /// Extended for projectiles: 300+ m/s
    pub max_velocity_ms: f32,
    
    /// Range resolution tradeoff (higher velocity = lower resolution)
    pub range_resolution_m: f32,
    
    /// Update rate (Hz) - higher for fast transients
    pub update_rate_hz: u32,
    
    /// CW mode for zero latency
    pub cw_mode: bool,
}

impl<SPI: SpiDevice> DopplerRadarDriver<SPI> {
    /// Configure radar for extended velocity range
    /// This trades range resolution for velocity range
    pub fn configure_extended(&mut self, max_velocity_ms: f32) -> Result<(), RadarError> {
        // Calculate chirp parameters for extended Doppler
        // ...
        self.max_velocity_ms = max_velocity_ms;
        Ok(())
    }
    
    /// Poll for high-velocity detection
    pub fn poll_high_velocity(&mut self) -> Option<HighVelocityDetection> {
        // Check for Doppler shifts corresponding to fast objects
        // ...
    }
}

pub struct HighVelocityDetection {
    pub velocity_ms: f32,
    pub bearing_rad: f32,
    pub range_m: f32,
    pub confidence: f32,
    pub timestamp_us: u64,
}
```

### 8.3 `camera_360.rs` — 360° Multi-Camera Array

```rust
/// 360° camera array driver
pub struct Camera360Driver {
    cameras: [CameraDriver; MAX_CAMERAS],
    camera_count: usize,
    fsync_gpio: GpioPin<Output<PushPull>>,
    calibration: CalibrationData,
    vision_processor: VisionProcessorInterface,
}

/// Calibration data for 360° stitching
#[derive(Clone, Debug, Deserialize)]
pub struct CalibrationData {
    /// Intrinsic parameters per camera
    pub intrinsics: Vec<CameraIntrinsics>,
    /// Extrinsic parameters (position relative to pendant center)
    pub extrinsics: Vec<CameraExtrinsics>,
    /// Stitching parameters
    pub stitching: StitchingConfig,
}

#[derive(Clone, Copy, Debug)]
pub struct CameraIntrinsics {
    pub focal_length_px: f32,
    pub principal_point: (f32, f32),
    pub distortion: [f32; 5],
}

#[derive(Clone, Copy, Debug)]
pub struct CameraExtrinsics {
    pub rotation: [[f32; 3]; 3],
    pub translation: [f32; 3],
}

impl Camera360Driver {
    /// Initialize all cameras
    pub fn init(&mut self) -> Result<(), CameraError>;
    
    /// Capture all cameras with hardware sync
    pub fn capture_sync(&mut self) -> Result<[Frame; MAX_CAMERAS], CameraError>;
    
    /// Load calibration from flash
    pub fn load_calibration(&mut self) -> Result<CalibrationData, StorageError>;
    
    /// Encode with tiered quality
    pub fn encode_tiered(
        &mut self,
        frames: &[Frame],
        tier: QualityTier,
        bandwidth_bps: u32,
    ) -> Result<EncodedStream, EncodeError>;
}

/// Quality tiers for bandwidth adaptation
#[derive(Clone, Copy, Debug)]
pub enum QualityTier {
    /// Instant baseline: QVGA from all cameras (~500 Kbps)
    Baseline,
    /// Standard: VGA from all cameras (~2-4 Mbps)
    Standard,
    /// High: 720p keyframes (~4-6 Mbps)
    High,
    /// Progressive: QVGA baseline + high-res keyframes
    Progressive,
}
```

### 8.4 `wifi.rs` — Belt Node Only

```rust
/// WiFi driver - BELT NODE ONLY
#[cfg(feature = "belt_node")]
pub struct WifiDriver {
    #[cfg(target_os = "linux")]
    interface: String,
    #[cfg(not(target_os = "linux"))]
    esp_at: EspAtDriver,
}

#[cfg(feature = "belt_node")]
impl WifiDriver {
    /// Connect to access point
    pub fn connect(&mut self, ssid: &str, password: &str) -> Result<(), WifiError>;
    
    /// Check if connected
    pub fn is_connected(&self) -> bool;
    
    /// Get local IP
    pub fn get_local_ip(&self) -> Option<Ipv4Addr>;
    
    /// Get signal strength
    pub fn get_rssi(&self) -> Option<i32>;
    
    /// Prefer 5 GHz band for BLE coexistence
    pub fn prefer_5ghz(&mut self, prefer: bool);
}
```

### 8.5 `cellular.rs` — Belt Node Only

```rust
/// Cellular SIM driver - BELT NODE ONLY
#[cfg(feature = "belt_node")]
pub struct CellularDriver<UART: SerialPort> {
    uart: UART,
    config: CellularConfig,
    registered: bool,
}

#[cfg(feature = "belt_node")]
impl<UART: SerialPort> CellularDriver<UART> {
    /// Power on cellular module
    pub fn power_on(&mut self) -> Result<(), CellularError>;
    
    /// Register to network
    pub fn register(&mut self) -> Result<(), CellularError>;
    
    /// Check if registered
    pub fn is_registered(&self) -> bool;
    
    /// Get signal strength (dBm)
    pub fn get_signal_strength(&mut self) -> Result<i32, CellularError>;
    
    /// Get carrier name
    pub fn get_carrier(&mut self) -> Result<String, CellularError>;
    
    /// Send AT command
    pub fn send_at(&mut self, cmd: &str) -> Result<String, CellularError>;
    
    /// Send alert to endpoint
    pub fn send_alert(&mut self, endpoint: &str, payload: &[u8]) -> Result<(), CellularError>;
    
    /// Check SIM presence
    pub fn check_sim(&mut self) -> Result<SimStatus, CellularError>;
}

pub enum SimStatus {
    Present { sim_type: SimType },
    NotPresent,
    Error,
}

pub enum SimType {
    NanoSim,
    ESIM,
}
```

### 8.6 `uwb.rs` — Optional on All Nodes

```rust
/// UWB driver for timing, ranging, and bandwidth augmentation
pub struct UwbDriver<SPI: SpiDevice> {
    spi: SPI,
    config: UwbConfig,
    role: UwbRole,
}

/// UWB role configuration
#[derive(Clone, Copy, Debug)]
pub enum UwbRole {
    /// Time synchronization only (lowest power)
    TimingOnly,
    /// Timing + ranging between nodes
    TimingAndRanging,
    /// Full bandwidth capability
    FullBandwidth,
}

#[derive(Clone, Copy, Debug)]
pub struct UwbConfig {
    /// UWB channel (1-9, typically 5 or 9 for UWB)
    pub channel: u8,
    /// Preamble length
    pub preamble_length: PreambleLength,
    /// Maximum sustained bandwidth target
    pub max_bandwidth_mbps: f32,
    /// Power mode
    pub power_mode: UwbPowerMode,
}

impl<SPI: SpiDevice> UwbDriver<SPI> {
    /// Initialize UWB
    pub fn init(&mut self) -> Result<(), UwbError>;
    
    /// Time synchronization exchange
    pub fn time_sync_exchange(&mut self) -> Result<TimeSyncResult, UwbError>;
    
    /// Measure range to another node
    pub fn measure_range(&mut self, target: NodeId) -> Result<f32, UwbError>;
    
    /// Send burst data
    pub fn send_burst(&mut self, data: &[u8]) -> Result<(), UwbError>;
    
    /// Start continuous stream (up to max_bandwidth_mbps)
    pub fn start_stream(&mut self) -> Result<(), UwbError>;
    
    /// Check if stream bandwidth is available
    pub fn check_bandwidth(&self) -> f32;
}

pub struct TimeSyncResult {
    pub offset_ns: i64,
    pub skew_ppm: f32,
    pub confidence: f32,
}
```

### 8.7 `storage.rs` — Any Node With SD Card

```rust
/// Storage driver for SD card with integrity chain
pub struct StorageDriver<SPI: SpiDevice> {
    spi: SPI,
    sd_card: SdCard,
    config: StorageConfig,
}

#[derive(Clone, Debug)]
pub struct StorageConfig {
    /// Maximum storage in MB (0 = unlimited)
    pub max_storage_mb: u64,
    /// Retention in days (0 = forever)
    pub retention_days: u32,
    /// Enable integrity chain
    pub enable_integrity: bool,
    /// Storage target
    pub target: StorageTarget,
}

impl<SPI: SpiDevice> StorageDriver<SPI> {
    /// Initialize SD card
    pub fn init(&mut self) -> Result<(), StorageError>;
    
    /// Open recording for writing
    pub fn open_recording(&mut self, path: &str) -> Result<RecordingWriter, StorageError>;
    
    /// List recordings
    pub fn list_recordings(&self) -> Result<Vec<RecordingMetadata>, StorageError>;
    
    /// Delete recording
    pub fn delete_recording(&mut self, path: &str) -> Result<(), StorageError>;
    
    /// Enforce retention policy
    pub fn enforce_retention(&mut self) -> Result<u64, StorageError>;
    
    /// Get used storage in MB
    pub fn get_used_mb(&self) -> u64;
}

/// Recording writer with integrity chain
pub struct RecordingWriter<'a> {
    file: File<'a>,
    hasher: Sha256,
    start_time_ms: u64,
}

impl<'a> RecordingWriter<'a> {
    /// Write frame to recording
    pub fn write_frame(&mut self, frame: &[u8]) -> Result<(), StorageError>;
    
    /// Close recording and generate integrity manifest
    pub fn close(self) -> Result<RecordingMetadata, StorageError>;
}

pub struct RecordingMetadata {
    pub path: String,
    pub start_time_ms: u64,
    pub duration_ms: u32,
    pub size_bytes: u64,
    pub sha256: [u8; 32],
    pub integrity_manifest: IntegrityManifest,
}
```

---

## 9. BAN Protocol Stack

### 9.1 Protocol Overview

All inter-node communication uses the Body-Area Network (BAN) protocol. External connectivity (WiFi, cellular) is handled exclusively by the belt node.

### 9.2 Message Definitions

```rust
/// All BAN messages
#[derive(Clone, Debug, Serialize, Deserialize)]
pub enum SentinelBanMessage {
    // === Sensor Data ===
    NodeSensorData {
        timestamp_ms: u64,
        node_id: NodeId,
        radar_metadata: RadarMetadata,
        imu_quaternion: [f32; 4],
        linear_accel: [f32; 3],
        audio_clip_available: bool,
        audio_clip_path: Option<String>,
    },
    
    // === Detections ===
    Detection {
        timestamp_ms: u64,
        node_id: NodeId,
        detection_type: DetectionType,
        position: Vector3,
        velocity: Vector3,
        confidence: f32,
        classification: ClassificationHint,
    },
    
    // === Gait Events ===
    GaitEvent {
        timestamp_ms: u64,
        node_id: NodeId,
        event_type: GaitEventType,
        phase: GaitPhase,
        confidence: f32,
        impact_g: Option<u8>,
    },
    
    // === Acoustic Events ===
    AcousticEvent {
        timestamp_ms: u64,
        node_id: NodeId,
        event_class: String,
        direction_of_arrival: (f32, f32),
        confidence: f32,
        audio_clip_path: Option<String>,
    },
    
    // === Identification ===
    IdentificationResult {
        timestamp_ms: u64,
        node_id: NodeId,
        classification: ClassificationTag,
        confidence: f32,
        raw_media_path: Option<String>,
        live_stream_available: bool,
        clip_duration_s: Option<f32>,
    },
    
    // === Extreme Velocity ===
    FastObjectDetection {
        timestamp_us: u64,
        node_id: NodeId,
        velocity_ms: f32,
        bearing_rad: f32,
        elevation_rad: f32,
        range_m: f32,
        confidence: f32,
    },
    
    // === Recording Lifecycle ===
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
    
    // === Media Streaming ===
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
    StreamEnd {
        node_id: NodeId,
        timestamp_ms: u64,
    },
    
    // === 360° Panorama ===
    PanoramaFrameAvailable {
        node_id: NodeId,
        timestamp_ms: u64,
        width: u32,
        height: u32,
        frame_path: Option<String>,
        quality_tier: QualityTier,
    },
    
    // === Alerts ===
    Alert {
        timestamp_ms: u64,
        alert_class: AlertClass,
        priority: AlertPriority,
        source_node: NodeId,
        description: String,
    },
    
    // === Haptic Commands ===
    HapticCommand {
        target_node: NodeId,
        pattern: HapticPattern,
        duration_ms: u16,
        intensity: u8,
    },
    
    // === Calibration ===
    CalibrationCommand {
        command: CalibrationPhase,
    },
    CalibrationData {
        node_id: NodeId,
        transform: BodyFrameTransform,
    },
    
    // === Time Synchronization ===
    TimeSyncExchange {
        t1_ns: u64,
        t2_ns: u64,
        t3_ns: u64,
        t4_ns: u64,
    },
    TimeSyncUpdate {
        offset_ns: i64,
        skew_ppm: f32,
    },
    
    // === Health ===
    NodeHealth {
        node_id: NodeId,
        battery_percent: u8,
        battery_mv: u16,
        charging: bool,
        temperature_c: f32,
        sensor_health: Vec<(String, SensorHealthStatus)>,
    },
    
    // === Configuration ===
    ConfigUpdate {
        node_id: NodeId,
        config_section: String,
        config_data: Vec<u8>,
    },
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum DetectionType {
    RadarPoint,
    LiDARCluster,
    AcousticSource,
    VisualObject,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum GaitEventType {
    HeelStrike,
    ToeOff,
    StumblePrecursor,
    FallDetected,
    GaitIrregularity,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum GaitPhase {
    Stance,
    Swing,
    Strike,
    Push,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum AlertClass {
    HumanApproaching,
    VehicleNear,
    GaitAnomaly,
    FallDetected,
    FastObjectDetected,
    SystemAlert,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum AlertPriority {
    Info,
    Warning,
    Critical,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum HapticPattern {
    SinglePulse,
    DoublePulse,
    TriplePulse,
    Sustained,
    RapidPulse,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum StreamFormat {
    H264,
    H265,
    MJPEG,
    RawRGB,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize)]
pub enum StreamQuality {
    Low,
    Medium,
    High,
    Raw,
}
```

### 9.3 Protocol Timing

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

## 10. Power Management

### 10.1 Power States

| State | CPU | Peripherals | BAN Radio | Wake Source | Power (Typical) |
|---|---|---|---|---|---|
| **Active** | Full | All on | Connected | N/A | 200-500 mW |
| **Idle** | WFI | Sensors active | Connected | Timer, interrupt | 50-100 mW |
| **Standby** | Off (RAM retained) | Key sensors | Advertising | Radar, timer | 5-20 mW |
| **Deep Sleep** | Off | RTC only | Off | External interrupt | <1 mW |

### 10.2 Belt Node Power States (Linux SoM)

| State | WiFi | Cellular | SLAM | Streaming | Power |
|---|---|---|---|---|---|
| **Minimal** | Idle | Off | Off | Off | ~500 mW |
| **Standard** | Connected | Standby | Off | Off | ~1.4 W |
| **Full Active** | Streaming | Active | On | On | ~4.5 W |
| **Remote Streaming** | Off | Streaming | On | On | ~5.3 W |

### 10.3 Power Profile Configuration

```rust
/// Power profile configuration
#[derive(Clone, Copy, Debug)]
pub struct PowerProfile {
    /// BLE connection interval (affects latency vs power)
    pub ble_interval: BleInterval,
    /// Sensor duty cycle (0.0-1.0)
    pub sensor_duty_cycle: f32,
    /// Camera power management
    pub camera_power: CameraPowerMode,
    /// UWB power mode
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

#[derive(Clone, Copy, Debug)]
pub enum CameraPowerMode {
    AlwaysOn,
    OnDemand,
    MotionTriggered,
    Off,
}

#[derive(Clone, Copy, Debug)]
pub enum UwbPowerMode {
    Disabled,
    TimingOnly,
    RangingActive,
    FullBandwidth,
}

impl From<&str> for PowerProfile {
    fn from(s: &str) -> Self {
        match s {
            "ultra_low_latency" => Self::ultra_low_latency(),
            "low_latency" => Self::low_latency(),
            "standard" => Self::standard(),
            "power_saving" => Self::power_saving(),
            _ => Self::standard(),
        }
    }
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
    
    pub fn power_saving() -> Self {
        Self {
            ble_interval: BleInterval::PowerSaving,
            sensor_duty_cycle: 0.25,
            camera_power: CameraPowerMode::Off,
            uwb_power: UwbPowerMode::Disabled,
        }
    }
}
```

---

## 11. Thermal Management

### 11.1 Temperature Monitoring

```rust
/// Thermal management for wearable nodes
pub struct ThermalManager {
    thermistor: NtcThermistor,
    throttling_threshold_c: f32,
    shutdown_threshold_c: f32,
    current_state: ThermalState,
}

#[derive(Clone, Copy, Debug)]
pub enum ThermalState {
    Normal,
    Throttled { reduction_percent: u8 },
    Critical,
}

impl ThermalManager {
    /// Check temperature and update state
    pub fn check(&mut self) -> ThermalState {
        let temp = self.thermistor.read_temperature();
        
        if temp > self.shutdown_threshold_c {
            self.current_state = ThermalState::Critical;
            // Initiate graceful shutdown
        } else if temp > self.throttling_threshold_c {
            let reduction = ((temp - self.throttling_threshold_c) / 5.0 * 100.0) as u8;
            self.current_state = ThermalState::Throttled {
                reduction_percent: reduction.min(50),
            };
        } else {
            self.current_state = ThermalState::Normal;
        }
        
        self.current_state
    }
    
    /// Get recommended SLAM frame rate based on thermal state
    pub fn get_slam_framerate(&self) -> u8 {
        match self.current_state {
            ThermalState::Normal => 30,
            ThermalState::Throttled { reduction_percent } => {
                (30 * (100 - reduction_percent) / 100) as u8
            }
            ThermalState::Critical => 0, // Disable SLAM
        }
    }
}
```

### 11.2 Temperature Thresholds

| Node | Throttling | Shutdown | Notes |
|------|------------|----------|-------|
| Pendant Standard | 42°C | 50°C | Skin contact limit |
| Pendant 360° | 40°C | 45°C | Higher thermal load |
| Belt MCU | 45°C | 55°C | Enclosure interior |
| Belt Linux SoM | 50°C | 60°C | With active cooling |
| Bracelet | 42°C | 50°C | Skin contact |
| Anklet | 45°C | 55°C | Less skin sensitivity |

---

## 12. Data Handling — User Configured

### 12.1 Configuration File

All data handling is user-configured in `sentinel-wear.toml`. The firmware imposes no restrictions.

```toml
# Data handling configuration
[data]
# Raw data storage
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
store_360_panorama = true

# Storage management
storage_target = "belt_sd_card"       # "belt_sd_card" | "node_local_sd" | "belt_internal"
retention_days = 30                    # 0 = keep forever
max_storage_mb = 32768                 # 0 = unlimited
continuous_recording = false           # Record continuously vs on-trigger
recording_trigger = "on_detection"     # "always" | "on_detection" | "on_alert" | "manual"

# Integrity
include_integrity_chain = true         # Generate SHA-256 manifests

# Streaming
enable_app_streaming = true
stream_quality = "high"                # "low" | "medium" | "high" | "raw"

# 360° pendant specific
[pendant_360]
progressive_quality = true
baseline_resolution = "qvga"
refinement_resolution = "720p"
adaptive_bandwidth = true

# Extreme velocity
[extreme_velocity]
enabled = true
mode = "production"                    # "disabled" | "research" | "production"
min_velocity_ms = 50
max_velocity_ms = 350
alert_on_detection = true

# Connectivity
[connectivity]
# WiFi - belt node only
wifi_enabled = true
wifi_ssid = "HomeNetwork"
wifi_prefer_5ghz = true

# Cellular - belt node only
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = "carrier.apn"
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""

# BAN
[connectivity.ban]
primary = "ble"                        # "ble" | "uwb" | "hybrid"
ble_interval = "standard"              # "ultra_low_latency" | "low_latency" | "standard" | "power_saving"
uwb_enabled = true
uwb_role = "timing_and_ranging"

# Power
[power]
profile = "standard"                   # "ultra_low_latency" | "low_latency" | "standard" | "power_saving"
thermal_throttling = true
shutdown_on_overheat = true

# App
[app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9092
enable_remote_access = false
remote_access_token = ""
max_concurrent_streams = 4
```

### 12.2 Live Streaming Flow

When the companion app requests a live stream from a pendant camera:

```
1. Companion app → Belt API: GET /api/nodes/pendant/stream
2. Belt firmware sends BAN message: StartStreamToBelt { node_id: pendant, format: H264, quality: High, use_uwb: true }
3. Pendant firmware:
   a. Starts camera capture
   b. Encodes frames to H.264
   c. Sends StreamData messages via BAN (BLE or UWB)
4. Belt firmware:
   a. Receives StreamData messages
   b. Relays to RTSP server on port 9090
   c. WiFi clients connect to rtsp://belt:9090/pendant
   d. Or cellular relay for remote access
5. Pendant never connects to WiFi or cellular
```

---

## 13. Build System

```toml
[workspace]
members = ["."]

[package]
name = "sentinel-wear-firmware"
version = "0.1.0"
edition = "2021"

[features]
default = []

# Hardware variants
belt_node = []
curved_360 = []
event_enhanced = []

# Optional hardware
uwb = []
hw_camera_switch = []

# Belt-only features
wifi = ["belt_node"]
cellular = ["belt_node"]

# Linux-only features (belt node SoM variants)
linux_som = ["belt_node"]
slam = ["linux_som"]
full_api = ["linux_som"]

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
# Embedded dependencies (no_std)
cortex-m = { version = "0.7", optional = true }
cortex-m-rt = { version = "0.7", optional = true }
embedded-hal = { version = "1.0" }
panic-halt = { version = "0.2", optional = true }
embedded-sdmmc = { version = "0.7" }
sha2 = { version = "0.10", default-features = false }
serde = { version = "1.0", default-features = false, features = ["derive"] }
postcard = { version = "1.0" }

# Linux dependencies (belt node SoM)
tokio = { version = "1.0", optional = true }
axum = { version = "0.7", optional = true }
tower = { version = "0.4", optional = true }
tracing = { version = "0.1", optional = true }

[target.'cfg(target_os = "linux")'.dependencies]
libc = "0.2"
```

---

## 14. Legal & Compliance

- **Sensing-only effectors:** No actuation path exists in any firmware binary.
- **Data handling:** User-configured. No firmware-imposed restrictions.
- **WiFi/Cellular:** Belt node firmware only. Radio modules require FCC/CE/ISED certification.
- **SAR compliance:** Critical for wearable devices. Use only body-worn certified BLE modules.
- **Recording laws:** Deployer's responsibility. Firmware does not enforce jurisdiction-specific restrictions.
- **Battery safety:** Hardware protections mandatory. Firmware provides secondary cutoff monitoring.
- **Integrity chain:** SHA-256 hashes generated for all recordings. Manifests stored with recordings.
- **Export control:** Encryption in firmware may be subject to export control regulations in some jurisdictions.

---

## 15. Debug & Test

### 15.1 Debug Interfaces

- **SWD:** 4-pin debug header (SWCLK, SWDIO, nRST, SWO)
- **RTT:** Non-intrusive logging via Segger RTT
- **UART:** Debug console at 115200 baud, 8N1
- **Test mode:** `std` build for host-side unit testing with mock `embedded-hal`

### 15.2 Test Build

```bash
# Build for hardware
cargo build --target thumbv7em-none-eabihf --bin pendant_node --release
cargo build --target aarch64-unknown-linux-gnu --bin belt_node_linux --release

# Build for testing (std target)
cargo test --features test
```

### 15.3 Firmware Verification Tests

| Test | Description | Pass Criteria |
|------|-------------|---------------|
| Power-on self-test | MCU boots, CRC verified | All sensors respond |
| BAN connectivity | Node connects to belt via BLE | < 200 ms association |
| Time sync | Node syncs clock with belt | Offset < 1 ms |
| Recording | Capture and store to SD | File exists with integrity manifest |
| Streaming | Start/stop stream via BAN | Frames arrive at belt |
| Thermal | Temperature monitoring | Throttling triggers at threshold |
| Cellular (belt only) | AT command response | Module responds < 5 s |
| WiFi (belt only) | AP association | Connects < 30 s |

---

## 16. Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2024-01 | Initial specification |
