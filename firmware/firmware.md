# Firmware Architecture — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V)

---

## 1. Purpose

This document specifies the firmware architecture for SENTINEL-WEAR's sensing nodes. The firmware runs on resource-constrained microcontrollers (MCUs) embedded in jewelry-form-factor wearables (pendant, bracelets, anklets, belt, eyewear).

The firmware is implemented in Rust using the `no_std` ecosystem. It is responsible for:
- Driving sensors (mmWave radar, IMU, LiDAR/ToF, acoustic, environmental, event, cameras).
- Performing node-local processing to reduce BAN (Body Area Network) bandwidth.
- Power management (critical for 24–72 hour battery life).
- Communicating with the belt controller over a low-power radio (BLE 5.x / UWB).
- Managing local storage (SD card, internal flash) for raw recordings per user configuration.
- Serving media streams to the companion app when configured.

**Scope Constraint:** This firmware is **sensing-and-information only**. It does not control actuators that act against external objects. The firmware does not impose any restrictions on what data users capture, store, or transmit from their own hardware. All data handling is user-configured.

---

## 2. Target Platforms

The reference architecture targets modern low-power MCUs with sufficient DSP capability for local signal processing.

**Primary Targets:**
- ARM Cortex-M4F / Cortex-M7 (e.g., STM32L4, STM32H7, nRF52840, nRF5340)
- RISC-V (e.g., ESP32-C6, BL808)

**Resource Profile (Minimum):**
- Flash: 512 KB (1 MB preferred)
- RAM: 256 KB (512 KB preferred for local buffering)
- DSP: FPU required; optional SIMD/Helium for advanced processing
- Storage: microSD card (for recording nodes); internal flash for logs and metadata
- Radios: BLE 5.x (mandatory), UWB (optional)

---

## 3. Workspace Structure

```
firmware/
├── Cargo.toml                # no_std workspace root
├── build.rs                  # Build script (features, configs)
├── memory.x                  # Linker script placeholder
├── firmware.md               # This document
├── README.md
└── src/
    ├── lib.rs                # Shared library code (drivers, logic)
    ├── bin/
    │   ├── pendant_node.rs
    │   ├── bracelet_left.rs
    │   ├── bracelet_right.rs
    │   ├── belt_node.rs
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
    │   ├── camera.rs
    │   └── storage.rs
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

## 4. Node Types and Profiles

Each node type is a separate binary (`src/bin/*.rs`), compiled with features enabled for its specific sensor set.

### 4.1 Belt Node (`belt_node.rs`)

**Role:** Primary compute unit, network coordinator, fusion hub, API server, recording manager.

**Hardware:**
- MCU: High-performance (Cortex-M7 or equivalent Linux-capable SoM)
- Sensors: mmWave radar (long-range, downward-facing), IMU (torso reference), Environmental.
- Storage: microSD card (primary recording destination), internal flash (metadata, logs).
- Radio: BLE + optional Wi-Fi / Cellular for remote connectivity.

**Firmware Responsibilities:**
- Run full multi-modal fusion (`sentinel-fusion` crate, if MCU permits).
- Route BAN traffic between all nodes.
- Run PentaTrack (`sentinel-tracking` crate) for body-frame prediction.
- Expose REST/WebSocket API for companion app (phone, PC).
- Manage recording lifecycle per `sentinel-wear.toml` data handling configuration.
- Serve live media streams to companion app over Wi-Fi when configured.
- Generate and store integrity manifests for all recordings.

**Power Strategy:**
- Highest power budget (largest battery).
- Active fusion during movement; low-power presence monitoring otherwise.
- Must support continuous operation of compute loop + BAN hub duties simultaneously.

---

### 4.2 Pendant Node — Standard & Medallion (`pendant_node.rs`)

**Role:** Upper-hemisphere sensing, identification, acoustic classification, visual capture (if configured).

**Hardware:**
- MCU: Mid-range (Cortex-M4F or higher for camera-enabled variants)
- Sensors: mmWave radar, IMU, Microphone array, Camera module (if configured).

**Firmware Responsibilities:**
- Detect upper-body threats via radar.
- Full-echo acoustic profiling and direction-of-arrival estimation.
- **Camera / Identification Logic (user-configured):**
  - Reads `DATA_HANDLING_MODE` from configuration at boot.
  - Metadata-only mode: on-device inference; transmit `IdentificationResult` only.
  - Local storage mode: capture raw frames/video to SD card per configured retention policy.
  - App streaming mode: stream live video to companion app over Wi-Fi via belt controller.
  - Full recording mode: store all data; generate integrity manifest per recording.
  - **Hardware power switch (if installed by user):** GPIO read at boot. If hardware switch present and reads DISABLED: no camera activation regardless of software config. If not installed: software config is the sole control.

**Power Strategy:**
- Event-triggered activation for cameras (if not set to continuous).
- Radar/IMU always-on at low duty cycle.

---

### 4.3 Pendant Node — 360° Curved (`pendant_node.rs` with `curved_360` feature)

**Role:** All standard pendant roles plus 360° continuous visual coverage.

**Additional Firmware Responsibilities:**
- Initialize and synchronize all N camera modules (hardware sync line).
- Multiplex camera streams: either stitch on-pendant (high-compute variant) or stream individual camera feeds to belt node for stitching.
- Store 360° panorama recordings to SD card when configured.
- Stream stitched equirectangular video to companion app when configured.
- Calibrate per-camera intrinsics from calibration JSON stored in flash.

**Power Management:**
- 360° mode: all cameras active consumes 2–5 W. Belt power supplementation supported.
- Event-triggered mode: cameras activated only when radar or event sensor detects motion.

---

### 4.4 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

**Role:** Forearm-frame coverage, directional alerts, optional visual capture.

**Hardware:**
- MCU: Low-power (Cortex-M4)
- Sensors: mmWave radar, IMU, Haptic actuator, Optional camera (Variant C).

**Firmware Responsibilities:**
- Radar: Forearm hemisphere scanning.
- IMU: Arm motion, gesture detection, body-frame pose contribution.
- Haptic: Execute patterns received from belt controller `HapticCommand` messages.
- Camera (Variant C, if configured): Arm-direction visual capture; stream or store per configuration.

**Power Strategy:**
- Aggressive sleep between polls.
- Haptic peak current management (back-EMF sensing for LRA resonance tracking).

---

### 4.5 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

**Role:** Ground-plane detection, gait analysis, optional ground-level visual capture.

**Hardware:**
- MCU: Low-power
- Sensors: LiDAR/ToF (short-range), IMU, Haptic, Optional downward camera (Variant B).

**Firmware Responsibilities:**
- ToF: Ground-level object detection and floor-clearance measurement.
- IMU: Gait pattern analysis (step detection, heel-strike, stumble precursor).
- Haptic: Directional alerts from below.
- Camera (Variant B, if configured): Ground-level recording per data handling config.

---

### 4.6 Eyewear Node (`eyewear_node.rs`)

**Role:** Forward field, high-speed event sensing, head-pose estimation, optional visual capture.

**Hardware:**
- MCU: Low-latency variant (Cortex-M4/M33)
- Sensors: Event sensor, IMU, Optional conventional camera (Variants B and C).

**Firmware Responsibilities:**
- Event sensor processing (microsecond latency spike-stream).
- Spike-stream trajectory extraction (fast transient detection).
- Forward-collision warning logic.
- IMU: Head-pose estimation for body-frame head/torso separation.
- Camera (Variants B/C): Forward visual capture; stream or store per configuration.
- SLAM contribution: visual odometry output for Variant C (full array).

---

## 5. Architecture Layers

### 5.1 HAL (Hardware Abstraction)

```rust
use embedded_hal::spi::SpiDevice;
use embedded_hal::digital::OutputPin;

pub struct MmWaveDriver<SPI, CS> {
    spi: SPI,
    cs: CS,
}
```

### 5.2 Driver Layer (`src/drivers/`)

**`mmwave.rs`:**
- Implements `poll_presence()` and `poll_velocity()`.
- Handles chirp configuration via SPI/UART.
- Micro-Doppler extraction for activity classification.

**`imu.rs`:**
- Polls at 200 Hz minimum.
- Runs local Madgwick filter.
- Outputs integrated orientation quaternions and gravity-compensated linear acceleration.

**`acoustic.rs`:**
- PDM/I2S read from microphone array.
- On-MCU delay-and-sum beamforming.
- Outputs direction-of-arrival and event-class metadata.
- Raw audio recording to SD card when `store_raw_audio = true` in config.

**`lidar_tof.rs`:**
- UART/SPI interface to ToF module.
- Outputs distance readings and 8×8 zone maps.

**`event_sensor.rs`:**
- High-bandwidth event stream ingestion via DMA.
- Spatiotemporal clustering for motion event extraction.
- Outputs `MotionEvent` metadata.

**`camera.rs`:**
- MIPI CSI-2 or DVP interface to camera module(s).
- Configures resolution, frame rate, and exposure based on data handling config.
- For 360° curved pendant: multiplexes N cameras; hardware sync via FSYNC line.
- Operates per user-configured data handling policy:
  - Metadata-only: runs local inference; outputs classification result only.
  - Local storage: captures frames/video to SD card via `storage.rs`.
  - App streaming: pushes compressed stream to belt node BAN radio for Wi-Fi relay.
  - Full recording: stores all frames with timestamps; generates integrity metadata.
- Hardware power switch support: reads optional GPIO at boot. If switch installed and reads DISABLED, returns to sleep without any camera init. If not installed, software config is sole control.

**`haptic.rs`:**
- I2C driver for DRV2605L haptic controller.
- LRA resonance tracking via back-EMF feedback.
- Pre-defined pattern library.

**`storage.rs`:**
- SPI interface to microSD card.
- FAT32 filesystem via embedded-sdmmc or equivalent.
- Functions: `open_recording()`, `write_frame()`, `close_recording()`, `list_recordings()`, `delete_recording()`.
- Retention enforcement: deletes oldest recordings when `max_storage_mb` limit reached.
- Integrity chain: generates SHA-256 hash of completed recording; stores manifest JSON alongside recording file.

### 5.3 Logic Layer (`src/logic/`)

**`body_frame_local.rs`:**
- Integrates local IMU data using Madgwick/Mahony filter.
- Estimates node orientation relative to torso reference.
- Provides `stabilize_detection()`.

**`presence_detection.rs`:**
- Radar presence polling state machine.
- Hysteresis filter to suppress clutter.
- Outputs `PresenceEvent` with classification hint.

**`recording_manager.rs`:**
- Manages recording lifecycle per `sentinel-wear.toml` data handling configuration.
- Handles triggers: always, on_detection, on_alert, manual, on_schedule.
- Coordinates with `storage.rs` for file management.
- Notifies belt controller of available recordings via `RecordingAvailable` BAN event.
- Streams media to belt controller for Wi-Fi relay when `enable_app_streaming = true`.

**`power_management.rs`:**
- Duty cycle control via RTOS task priorities.
- Sleep state entry/exit based on interrupt activity.
- Battery level monitoring via fuel gauge IC (I2C).
- Reports `BatteryStatus` to belt controller.

---

## 6. BAN Protocol Stack (`src/ban_radio/`)

The Body Area Network protocol is optimized for:
- Low latency (sub-millisecond event alerts to haptic output).
- Flexible data transmission (metadata, compressed clips, full streams).
- Secure pairing (challenge-response at commissioning time).

**Message Types:**

```rust
pub enum SentinelBanMessage {
    NodeSensorData {
        timestamp_ms: u64,
        radar_metadata: RadarMetadata,
        imu_quaternion: [f32; 4],
        linear_accel: [f32; 3],
        node_id: NodeId,
        audio_clip_available: bool,
        audio_clip_path: Option<String>,
    },
    HapticCommand {
        pattern: HapticPattern,
        duration_ms: u16,
        intensity: u8,
    },
    IdentificationResult {
        classification: ClassificationTag,
        confidence: f32,
        node_id: NodeId,
        raw_media_path: Option<String>,
        live_stream_available: bool,
        clip_duration_s: Option<f32>,
        still_image_path: Option<String>,
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
    },
    MediaStreamAvailable {
        node_id: NodeId,
        stream_port: u16,
        format: String,
    },
    MediaStreamEnded {
        node_id: NodeId,
        timestamp_ms: u64,
    },
    PanoramaFrameAvailable {
        node_id: NodeId,
        timestamp_ms: u64,
        width: u32,
        height: u32,
        frame_path: Option<String>,
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

## 7. Data Handling — Fully User Configurable

All data storage and transmission is user-configured in `sentinel-wear.toml`. The firmware enforces the user's configuration. It does not impose any restrictions on what users capture from their own hardware.

**Supported modes (configured in `sentinel-wear.toml`):**

```toml
[data]
store_raw_video = false               # Store raw video from camera nodes
store_raw_audio = false               # Store raw audio from microphone nodes
store_raw_sensor_data = false         # Store raw radar/ToF data
store_360_panorama = false            # Store 360° panoramic recordings
storage_target = "belt_sd_card"       # "belt_sd_card" | "node_local_sd" | "companion_app"
retention_days = 30                   # 0 = keep forever
continuous_recording = false          # Record continuously vs on-trigger
recording_trigger = "on_detection"    # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 32768               # 0 = unlimited
include_integrity_chain = true        # Include SHA-256 + chain hash for legal export

[app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9091
enable_remote_access = false
remote_access_token = ""
```

**Metadata-only mode (default — no raw storage configured):**
- On-device inference runs locally.
- Only classification results and detection metadata transmitted over BAN.
- Minimum bandwidth usage.

**Local storage mode (`store_raw_video = true`):**
- Raw frames, video clips, and audio stored to SD card per retention policy.
- User accesses recordings via companion app or by physically removing SD card.
- Suitable for legal evidence, security investigations, system calibration.

**App streaming mode (`enable_app_streaming = true`):**
- Live video/audio streamed to companion app over Wi-Fi via belt node.
- User views live footage and reviews recordings from mobile or desktop.

**Full recording mode (`continuous_recording = true`):**
- All sensor data stored continuously with timestamped integrity chain.
- Available for playback, export, and legal proceedings.

**Privacy controls (all user-selected — none system-imposed):**
- Hardware power switch: optional user-installed switch that cuts power to camera module. If installed, read at boot; if DISABLED, camera remains inactive regardless of software config.
- Software scheduling: camera inactive during configured hours.
- Detection-triggered mode: camera activates only on motion/detection event.
- Manual mode: camera activates only on explicit user command.
- Any combination of the above.

---

## 8. Power Management Strategy

| State | CPU | Peripherals | Radio | Wake Source |
|---|---|---|---|---|
| **Active** | Full speed | All on | Connected | N/A |
| **Idle** | WFI halted | Sensors active | Connected | Timer, interrupt |
| **Standby** | Off (RAM retained) | Key sensors on | Advertising | Radar, timer |
| **Deep Sleep** | Off | RTC only | Off | External interrupt only |

### 8.1 Node Lifecycle

1. Boot & Init: Peripherals up, check optional hardware switches, read fuel gauge.
2. Read `sentinel-wear.toml` data handling configuration from flash.
3. Pairing/Commission: Await pairing command from belt; exchange security credentials.
4. Calibration: Await `CalibrationCommand` from belt; stream high-rate IMU data.
5. Normal Operation: Poll sensors per duty cycle; transmit per configuration; enter Idle or Standby.
6. Event Mode: Detection triggers immediate priority wake and transmit.
7. Low Battery: Reduce sensor duty cycles; alert belt controller; suspend recording.

---

## 9. Camera Node Architecture (All Camera-Capable Nodes)

### 9.1 Boot Sequence

1. `main()` initializes GPIO, RTC, and I2C/SPI buses.
2. **Optional hardware switch check:** If `CAMERA_HW_SWITCH_GPIO` pin is configured in hardware:
   - Read GPIO state.
   - If DISABLED: log status; do not initialize camera driver; camera remains off.
   - If ENABLED or pin not configured: proceed to step 3.
3. Read data handling configuration from `sentinel-wear.toml`.
4. Initialize camera driver per selected data handling mode:
   - Metadata-only: configure camera for inference-optimized settings.
   - Storage mode: configure camera for quality settings matching storage config.
   - Streaming mode: configure encoder for streaming quality setting.
   - Full recording: configure camera for full-quality capture.
5. Await activation command from belt controller.
6. Begin capture per configured mode.

### 9.2 Data Handling Per Mode

**Metadata-only:**
- Inference on-MCU or on dedicated NPU module.
- Transmit `IdentificationResult` over BAN.
- No frame buffering after inference.

**Local storage:**
- Frames written to SD card via `storage.rs`.
- SHA-256 computed over completed recording.
- Integrity manifest generated and stored alongside recording.
- `RecordingAvailable` BAN event sent to belt controller.

**App streaming:**
- Compressed video (H.264 or MJPEG) encoded on-MCU or on ISP.
- Transmitted to belt controller over BAN for Wi-Fi relay to companion app.
- Simultaneously stored locally if `store_raw_video = true`.

**Full recording:**
- Raw or minimally-compressed frames stored continuously.
- All data available for review, export, and legal use.
- Belt controller receives `RecordingAvailable` events for each completed segment.

---

## 10. Build System

### `Cargo.toml` (Firmware Workspace)

```toml
[workspace]
resolver = "2"
members = ["."]

[package]
name = "sentinel-wear-firmware"
version = "0.1.0"
edition = "2021"

[lib]
name = "sentinel_firmware"
path = "src/lib.rs"
test = false
bench = false

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
embedded-hal = "1.0"
panic-halt = "0.2"
embedded-sdmmc = "0.7"

[features]
default = []
curved_360 = []        # Enable 360° multi-camera pendant support
uwb = []               # Enable UWB time sync
hw_camera_switch = []  # Enable hardware power switch GPIO check

[[bin]]
name = "pendant_node"
path = "src/bin/pendant_node.rs"

[[bin]]
name = "pendant_node_360"
path = "src/bin/pendant_node.rs"
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

### `build.rs`

```rust
fn main() {
    println!("cargo:rustc-link-arg=-Tlink.x");
    println!("cargo:rerun-if-changed=link.x");
    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rerun-if-changed=sentinel-wear.toml");
}
```

---

## 11. Legal & Compliance

- **Sensing-Only Effectors:** This firmware does not control actuators that act against external objects. No actuator output path exists beyond the haptic motor (wearer-facing only).
- **Data Handling:** All data capture, storage, and transmission is user-configured. The system does not impose any system-level restrictions. Jurisdiction-specific compliance with recording consent laws, privacy regulations, and data protection laws is the deployer's responsibility.
- **Hardware Privacy Switch:** Optional. Available as a hardware feature for users who want unconditional physical camera disable. Not required by the architecture.
- **Legal Evidence:** Recordings include embedded integrity metadata (SHA-256 hash, device ID, firmware version, timestamp chain). Export via companion app produces a manifest suitable for legal proceedings.
- **Export Control:** No controlled sensing parameters. Research-phase system.
- **Radio:** FCC Part 15 (US), CE RED (EU) compliance required for BLE radio. Ensure radio module is certified.
- **Battery:** Firmware enforces voltage cutoffs (4.25 V overcharge, 3.0 V overdischarge). Hardware protections are primary; firmware is secondary.

---

## 12. Companion App Architecture

SENTINEL-WEAR provides mobile (iOS/Android) and desktop (Windows/macOS/Linux) companion apps. The belt node serves as the gateway, running an embedded HTTP/WebSocket server accessible over Wi-Fi (home network) or Bluetooth (direct).

### 12.1 Mobile App (iOS / Android)

- **Live Sensor View:** Real-time body-frame display with node positions, detection vectors, PentaTrack prediction centers, alert zones, and entity classification.
- **Live Camera / Sensor Feed:** Stream live video or audio from any camera-equipped node on demand.
- **360° Live View:** Stream equirectangular 360° panorama from curved pendant cameras if installed.
- **Recordings Library:** Browse all stored recordings by date, node, trigger, or event type. Full playback.
- **Alert History:** Complete log of all detection, gait, acoustic, and identification events.
- **Gait Analytics:** Gait pattern history, stride analysis, stumble event log.
- **System Configuration:** Node management, sensor settings, data handling policy, BAN configuration.
- **Remote Access:** Optionally access the system from outside the home network when user configures token.
- **Legal Export:** Export any recording with cryptographic integrity manifest.

### 12.2 Desktop App (Windows / macOS / Linux)

- **Multi-Node Recording Management:** Timeline view across all nodes. Search, annotate, tag, and export.
- **3D World Model Replay:** Time-scrub through the SLAM world model (Linux SoM belt variant).
- **Body-Frame Analytics:** Visualize where detections occurred relative to wearer's body over time.
- **Gait Reports:** Long-term stride analysis, asymmetry history, anomaly log.
- **Configuration Manager:** Full TOML config editing.
- **Legal Export Suite:** Export clips with full integrity chain. SHA-256 hash, device ID, timestamp, firmware version. Suitable for legal proceedings and insurance documentation.
- **Bulk Export / Backup:** Export date ranges; schedule automatic archiving.

### 12.3 Data Flow

```
Node Sensors (all nodes)
        │
        ▼
BAN (BLE 5.x / UWB)
        │
        ▼
Belt Node (Fusion + API Server + Recording Manager)
        │
        ├──► Local Storage (SD card / Internal Flash)
        │         (raw video, audio, metadata, panoramas, sensor data)
        │
        └──► Wi-Fi / BLE
                  │
                  ├──► Mobile Companion App (live view, alerts, recordings)
                  └──► Desktop Companion App (full management, export, SLAM review)
```

### 12.4 Belt Node API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/nodes` | All nodes with health, capabilities, battery |
| GET | `/api/detections/recent` | Recent detection events |
| GET | `/api/alerts` | Alert history (paginated) |
| GET | `/api/recordings` | List recordings (paginated, filterable) |
| GET | `/api/recordings/{id}` | Download recording file |
| GET | `/api/recordings/{id}/meta` | Integrity manifest JSON |
| DELETE | `/api/recordings/{id}` | Delete recording and manifest |
| GET | `/api/export/{id}` | Export with integrity chain |
| GET | `/api/nodes/{id}/stream` | Open live media stream |
| GET | `/api/nodes/{id}/panorama` | Open live 360° panorama stream |
| GET | `/api/world-model` | Current world model state |
| GET | `/api/gait/history` | Gait event history |
| GET | `/api/analytics/body-frame` | Body-frame activity analytics |
| POST | `/api/config` | Update system configuration |
| GET | `/api/config` | Read current configuration |
| POST | `/api/policy/mode` | Change operational mode |

### 12.5 WebSocket Streams

| Path | Description |
|---|---|
| `/ws/events` | All live SENTINEL-WEAR events |
| `/ws/detections` | Live detection stream |
| `/ws/gait` | Live gait events |
| `/ws/alerts` | Live alert events |
| `/ws/world-model` | Live world model updates |

### 12.6 Media Streaming Ports

| Protocol | Port | Description |
|---|---|---|
| RTSP | 9090 | Per-node video stream |
| WebSocket H.264 | 9091 | Browser/app-compatible stream |
| WebSocket 360° | 9092 | Equirectangular panorama stream |
| PCM Audio | 9093 | Per-node raw audio stream |

### 12.7 Configuration (`sentinel-wear.toml`)

```toml
[data]
store_raw_video = false
store_raw_audio = false
store_360_panorama = false
store_raw_sensor_data = false
storage_target = "belt_sd_card"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 32768
include_integrity_chain = true

[app]
enabled = true
bind_address = "0.0.0.0"
api_port = 8080
media_port = 9090
panorama_port = 9092
enable_remote_access = false
remote_access_token = ""
max_concurrent_streams = 4
stream_quality = "medium"
```

---

## 13. Debug & Test

**On-Target Debug:**
- SWD (Serial Wire Debug) via 4-pin debug header.
- RTT (Real Time Transfer) for non-intrusive logs.
- UART console at 115200 baud for boot diagnostics.

**Host Simulation (`std` build):**
- `src/lib.rs` builds as `std` for host-side unit testing.
- Mock `embedded-hal` implementations provided for CI.
- Every driver has a `Simulated*` variant (mirrors OMNI-SENSE driver pattern).
- Recording manager testable with mock filesystem.
