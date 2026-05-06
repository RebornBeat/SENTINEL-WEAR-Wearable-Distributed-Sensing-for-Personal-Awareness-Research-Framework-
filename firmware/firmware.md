# Firmware Architecture — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V)

---

## 1. Purpose

This document specifies the firmware architecture for SENTINEL-WEAR's sensing nodes. The firmware runs on resource-constrained MCUs embedded in jewelry-form-factor wearables (pendant, bracelets, anklets, belt, eyewear).

The firmware is implemented in Rust using the `no_std` ecosystem. It is responsible for:
- Driving sensors (mmWave radar, IMU, LiDAR/ToF, acoustic, environmental, event, cameras).
- Performing node-local processing to reduce BAN bandwidth.
- Power management (critical for 24–72 hour battery life).
- **For all non-belt nodes:** communicating with the belt controller over BAN (BLE 5.x or UWB) only. No WiFi, no cellular.
- **For the belt node:** managing BAN hub, WiFi connectivity, optional cellular SIM, companion app API server, and recording lifecycle.

**Scope Constraint:** This firmware is sensing-and-information only. No effectors. Data handling is user-configured without system-imposed restrictions.

---

## 2. Connectivity Architecture in Firmware

**Non-belt node firmware (pendant, bracelets, anklets, eyewear):**
- BAN radio driver: BLE 5.x stack (or UWB on nodes with DW3000)
- No WiFi driver
- No cellular driver
- Data flow: sensors → local processing → BAN transmission to belt

**Belt node firmware:**
- BAN hub: receives from all nodes, coordinates time-sync
- WiFi driver: via SoM (Linux SoM) or ESP32 co-processor
- Cellular driver: Quectel AT-command layer via UART or USB
- App server: axum HTTP/WebSocket (Linux SoM) or minimal HTTP (MCU variant)
- Recording manager: aggregates all node recording events, manages belt SD card

**All data that reaches the companion app routes through the belt node.** When a pendant camera encodes video and transmits it, the flow is:
```
Pendant → [BAN] → Belt → [WiFi or Cellular] → Companion App
```

---

## 3. Target Platforms

**Primary:**
- ARM Cortex-M4F / Cortex-M7 (STM32L4, STM32H7, nRF52840, nRF5340)
- RISC-V (ESP32-C6, BL808)

**Belt node (full capability):**
- Linux SoM: Raspberry Pi CM4, NXP i.MX 8M Plus, Qualcomm SA8155P

**Resource Profile (Minimum — Non-Belt):**
- Flash: 512 KB (1 MB preferred)
- RAM: 256 KB
- Storage: microSD for recording-capable nodes

---

## 4. Workspace Structure

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
    │   ├── pendant_node.rs         # Variants A, C (standard + medallion)
    │   ├── pendant_360.rs          # Variant B (360° curved)
    │   ├── bracelet_left.rs
    │   ├── bracelet_right.rs
    │   ├── belt_node.rs            # All belt variants
    │   ├── anklet_left.rs
    │   ├── anklet_right.rs
    │   └── eyewear_node.rs
    ├── drivers/
    │   ├── mod.rs
    │   ├── mmwave.rs
    │   ├── imu.rs
    │   ├── acoustic.rs
    │   ├── lidar_tof.rs
    │   ├── event_sensor.rs
    │   ├── environmental.rs
    │   ├── haptic.rs
    │   ├── camera.rs               # Non-belt nodes with cameras
    │   ├── camera_360.rs           # 360° curved pendant
    │   ├── storage.rs              # SD card + integrity chain
    │   ├── wifi.rs                 # Belt node only
    │   └── cellular.rs             # Belt node only
    ├── logic/
    │   ├── mod.rs
    │   ├── body_frame_local.rs
    │   ├── presence_detection.rs
    │   ├── recording_manager.rs
    │   └── power_management.rs
    └── ban_radio/
        ├── mod.rs
        └── protocol.rs
```

---

## 5. Node Binary Descriptions

### 5.1 Belt Node (`belt_node.rs`)

**Role:** Primary compute, BAN hub, **sole external network gateway**, API server, recording manager.

**Drivers used:** `mmwave.rs`, `imu.rs`, `environmental.rs`, `storage.rs`, **`wifi.rs`**, **`cellular.rs`**

**Firmware responsibilities:**
- BAN hub: coordinates all inter-node communication, time-sync master
- WiFi connectivity: connects to home network; serves companion app API
- Cellular SIM: maintains LTE/5G connection when configured; forwards alerts when away
- App server: HTTP/WebSocket API for companion app
- Recording manager: aggregates `RecordingAvailable` events from all nodes; manages belt SD storage; serves recordings via API; generates integrity manifests
- Sensor fusion coordination: routes all node data to Rust crate stack for processing

**No other node has WiFi or cellular drivers.**

### 5.2 Pendant Node — Standard/Medallion (`pendant_node.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, `acoustic.rs`, `camera.rs` (if configured), `storage.rs` (if configured)

**Key behaviors:**
- Camera data handling per `sentinel-wear.toml`:
  - Metadata-only: inference on-MCU; transmit `IdentificationResult` via BAN
  - Local storage: frames → SD card; `RecordingAvailable` → BAN → belt
  - Belt relay: encoded video → BAN → belt → WiFi/cellular → app
- Optional hardware switch: if GPIO configured, read at boot; if DISABLED, camera uninit
- All camera-related data exits pendant via BAN to belt only

### 5.3 Pendant Node — 360° Curved (`pendant_360.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, `acoustic.rs`, `camera_360.rs`, `storage.rs`

**Additional responsibilities:**
- Initialize N camera modules simultaneously
- Generate FSYNC hardware sync pulse
- Load calibration JSON from flash at boot
- Transmit raw camera streams or compressed 360° video via BAN to belt
- Belt node handles stitching and relays to companion app via WiFi/cellular

### 5.4 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

**Drivers:** `mmwave.rs`, `imu.rs`, optional `camera.rs`

- Radar, IMU, haptic, optional camera
- All data → BAN → belt

### 5.5 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

**Drivers:** `lidar_tof.rs`, `imu.rs`, optional `camera.rs`

- ToF/LiDAR, IMU, haptic, gait analysis, optional camera
- All data → BAN → belt

### 5.6 Eyewear Node (`eyewear_node.rs`)

**Drivers:** `event_sensor.rs`, `imu.rs`, optional `camera.rs`

- Event sensor, IMU, optional camera(s)
- All data → BAN → belt

---

## 6. Driver Layer

### `wifi.rs` (Belt Node Only)

```rust
// Belt node WiFi management
pub struct WifiDriver {
    // SoM-specific (CM4: uses Linux NetworkManager; MCU: ESP-AT or custom)
}
impl WifiDriver {
    pub fn connect_to_ap(&mut self, ssid: &str, password: &str) -> Result<(), WifiError>;
    pub fn get_local_ip(&self) -> Option<Ipv4Addr>;
    pub fn is_connected(&self) -> bool;
}
```

### `cellular.rs` (Belt Node Only)

```rust
// Belt node cellular SIM management (Quectel AT command interface)
pub struct CellularDriver {
    uart: impl SerialPort,
    pub config: CellularConfig,
}
impl CellularDriver {
    pub fn power_on(&mut self) -> Result<(), CellularError>;
    pub fn register_network(&mut self) -> Result<(), CellularError>;
    pub fn get_signal_strength(&mut self) -> Result<i32, CellularError>; // dBm
    pub fn send_alert(&mut self, endpoint: &str, payload: &[u8]) -> Result<(), CellularError>;
    pub fn is_connected(&self) -> bool;
    pub fn get_carrier(&mut self) -> Result<String, CellularError>;
}
```

### `camera.rs` (Any Node With Camera)

- MIPI CSI-2 or DVP interface
- Configured per data handling mode
- Hardware power switch check at boot (if GPIO configured)
- Operates per user-configured mode:
  - Metadata-only: inference on-MCU; result → BAN
  - Local storage: frames → SD card (via `storage.rs`)
  - Belt relay: encoded stream → BAN → belt handles WiFi/cellular relay

### `camera_360.rs` (360° Pendant Only)

- Multiplexes N camera MIPI lanes
- Generates FSYNC hardware sync to all cameras simultaneously
- Loads calibration JSON from flash at boot
- Streams raw or compressed frames → BAN → belt (stitching on belt)

### `storage.rs` (Any Node With SD Card)

- SPI/SDMMC microSD via `embedded-sdmmc`
- FAT32 filesystem
- SHA-256 hash on completed recordings
- Integrity manifest generation
- `RecordingAvailable` BAN event on completion
- Retention enforcement per `max_storage_mb`

---

## 7. BAN Protocol Stack (`src/ban_radio/`)

**All inter-node communication is BAN.** WiFi and cellular exist only at the belt.

```rust
pub enum SentinelBanMessage {
    NodeSensorData {
        timestamp_ms: u64,
        radar_metadata: RadarMetadata,
        imu_quaternion: [f32; 4],
        linear_accel: [f32; 3],
        node_id: NodeId,
        audio_clip_available: bool,         // Raw audio on node SD card
        audio_clip_path: Option<String>,    // Path on node storage
    },
    HapticCommand {
        pattern: HapticPattern,
        duration_ms: u16,
        intensity: u8,                       // 0–100
    },
    IdentificationResult {
        classification: ClassificationTag,
        confidence: f32,
        node_id: NodeId,
        raw_media_path: Option<String>,      // Path on node SD card
        live_stream_available: bool,          // Stream available via BAN→belt→WiFi
        clip_duration_s: Option<f32>,
        still_image_path: Option<String>,
    },
    RecordingAvailable {
        node_id: NodeId,
        recording_path: String,              // Path on node SD card
        start_timestamp_ms: u64,
        duration_ms: u32,
        size_bytes: u64,
        trigger: String,
        format: String,
        modalities: Vec<String>,
        sha256: String,
    },
    // Belt node requests this node to start streaming to belt via BAN
    StartStreamToBelt {
        node_id: NodeId,
        format: String,                      // Belt will relay via WiFi/cellular to app
        quality: StreamQuality,
    },
    StopStreamToBelt {
        node_id: NodeId,
    },
    PanoramaFrameAvailable {
        node_id: NodeId,
        timestamp_ms: u64,
        width: u32,
        height: u32,
        frame_path: Option<String>,          // On node SD if stored; otherwise BAN-transmitted
    },
    GaitEvent {
        event_type: GaitEventType,
        phase: GaitPhase,
        confidence: f32,
        impact_g: Option<u8>,
    },
    AcousticEvent {
        event_class: String,
        direction_of_arrival: (f32, f32),
        confidence: f32,
        audio_clip_path: Option<String>,
    },
    BatteryStatus {
        percent: u8,
        voltage_mv: u16,
        charging: bool,
    },
    CalibrationCommand {
        command: CalibrationPhase,
    },
    NodeHealth {
        node_id: NodeId,
        sensor_health: Vec<(String, SensorHealthStatus)>,
    },
    TimeSyncExchange {
        t1_ns: u64,
        t2_ns: u64,
        t3_ns: u64,
        t4_ns: u64,
    },
}
```

---

## 8. Data Handling — User Configured

All data storage and transmission user-configured in `sentinel-wear.toml`. No restrictions imposed by firmware.

```toml
[data]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
store_360_panorama = false
storage_target = "belt_sd_card"    # "belt_sd_card" | "node_local_sd" | "belt_internal"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 32768
include_integrity_chain = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""
emergency_contact_trigger = "manual"

[app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9091
enable_remote_access = false
remote_access_token = ""
max_concurrent_streams = 4
```

**How live streaming works (example: pendant camera):**
1. Belt controller sends `StartStreamToBelt` BAN command to pendant.
2. Pendant encodes video, sends compressed frames via BAN to belt node.
3. Belt node receives BAN frames, relays via WiFi (or cellular if configured) to companion app.
4. Companion app displays live video via RTSP or WebSocket stream from belt server.

The pendant never connects to WiFi or cellular. The belt is the relay point.

---

## 9. Power Management

| State | CPU | Peripherals | BAN Radio | Wake Source |
|---|---|---|---|---|
| **Active** | Full | All on | Connected | N/A |
| **Idle** | WFI | Sensors active | Connected | Timer, interrupt |
| **Standby** | Off (RAM retained) | Key sensors | Advertising | Radar, timer |
| **Deep Sleep** | Off | RTC only | Off | External interrupt |

### Belt Node Power States

The belt node has additional states due to WiFi and cellular:
- **App Connected:** WiFi associated, API server active, cellular optional standby
- **Away Mode:** WiFi disconnected, cellular active for alert delivery
- **Emergency:** Cellular active, GPS (if available) enabled, emergency contact reachable

---

## 10. Build System

```toml
[workspace]
members = ["."]

[package]
name = "sentinel-wear-firmware"
version = "0.1.0"
edition = "2021"

[features]
default = []
curved_360 = []
uwb = []
hw_camera_switch = []
wifi = []          # Belt node only
cellular = []      # Belt node only

[[bin]]
name = "pendant_node"
path = "src/bin/pendant_node.rs"

[[bin]]
name = "pendant_node_360"
path = "src/bin/pendant_360.rs"
required-features = ["curved_360"]

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
name = "anklet_left"
path = "src/bin/anklet_left.rs"

[[bin]]
name = "anklet_right"
path = "src/bin/anklet_right.rs"

[[bin]]
name = "eyewear_node"
path = "src/bin/eyewear_node.rs"
```

---

## 11. Legal & Compliance

- **Sensing-only effectors:** No actuation path exists.
- **Data handling:** User-configured. No restrictions.
- **WiFi/Cellular:** Belt node only. FCC/CE/ISED certification required for radio modules.
- **SAR compliance:** Critical for wearable devices. Use only body-worn certified BLE modules.
- **Recording laws:** Deployer's responsibility.
- **Battery:** Hardware protections mandatory; firmware secondary cutoff.

---

## 12. Companion App Architecture

### 12.1 Architecture Summary

The belt node runs an embedded HTTP/WebSocket server. All data from all nodes routes through the belt first.

```
Node Sensors (all nodes)
       │
       ▼
BAN (BLE 5.x / UWB) — all inter-node traffic
       │
       ▼
Belt Node
  ├── Fusion + PentaTrack processing
  ├── Recording management (from all node SD cards + belt SD)
  └── App Server (API + media)
       │
       ├── WiFi (home network) ──► Companion App (local)
       └── Cellular SIM ──────► Companion App (remote)
                           └──► Emergency Contact
```

### 12.2 Mobile App Features

- Live sensor view (body-centric radar display)
- Live camera feed from any node (routed via belt WiFi/cellular)
- 360° live view from curved pendant (stitched on belt, relayed via WiFi/cellular)
- Recordings library (stored on belt SD or node SD cards)
- Gait analytics
- System configuration
- Remote access (via cellular when configured)
- Legal export with integrity chain

### 12.3 Belt Node API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/nodes` | All nodes + health |
| GET | `/api/detections/recent` | Recent detections |
| GET | `/api/alerts` | Alert history |
| GET | `/api/recordings` | All recordings |
| GET | `/api/recordings/{id}` | Download recording |
| GET | `/api/recordings/{id}/meta` | Integrity manifest |
| DELETE | `/api/recordings/{id}` | Delete recording |
| GET | `/api/export/{id}` | ZIP: recording + manifest |
| GET | `/api/nodes/{id}/stream` | Open live media stream |
| GET | `/api/nodes/{id}/panorama` | Open 360° panorama stream |
| GET | `/api/world-model` | Current world model |
| GET | `/api/gait/history` | Gait history |
| POST | `/api/config` | Update config |
| GET | `/api/config` | Read config |
| GET | `/api/connectivity` | WiFi + cellular status |
| GET | `/api/connectivity/cellular` | Cellular signal, carrier, data usage |
| POST | `/api/connectivity/cellular/toggle` | Enable/disable cellular |

### 12.4 WebSocket Streams

| Path | Description |
|---|---|
| `/ws/events` | All live BAN events |
| `/ws/detections` | Live detections |
| `/ws/gait` | Live gait events |
| `/ws/alerts` | Live alerts |
| `/ws/world-model` | Live world model updates |

### 12.5 Media Streaming Ports (Belt Node)

| Protocol | Port | Description |
|---|---|---|
| RTSP | 9090 | Per-node video (belt relays from BAN) |
| WebSocket H.264 | 9091 | Browser/app stream |
| WebSocket 360° | 9092 | Equirectangular panorama (stitched on belt) |
| PCM Audio | 9093 | Per-node audio (belt relays from BAN) |

### 12.6 Configuration (`sentinel-wear.toml`)

```toml
[data]
store_raw_video = false
store_raw_audio = false
store_360_panorama = false
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 32768
include_integrity_chain = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""

[app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9092
enable_remote_access = false
remote_access_token = ""
```

---

## 13. Debug & Test

- SWD via 4-pin debug header
- RTT for non-intrusive logs
- UART console at 115200 baud
- `std` build for host-side unit testing with mock `embedded-hal`
