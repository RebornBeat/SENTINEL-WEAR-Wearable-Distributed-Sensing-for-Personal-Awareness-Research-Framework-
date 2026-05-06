# Firmware Architecture — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research | **Target:** `no_std` (ARM Cortex-M / RISC-V)

---

## 1. Purpose

This document specifies the firmware architecture for SENTINEL-WEAR's sensing nodes. The firmware runs on resource-constrained microcontrollers (MCUs) embedded in jewelry-form-factor wearables (pendant, bracelets, anklets, belt, eyewear).

The firmware is implemented in Rust using the `no_std` ecosystem. It is responsible for:
- Driving sensors (mmWave radar, IMU, LiDAR/ToF, acoustic, environmental, event, identification).
- Performing node-local processing to reduce BAN (Body Area Network) bandwidth.
- Power management (critical for 24–72 hour battery life).
- Communicating with the belt controller over a low-power radio (BLE 5.x / UWB).

**Scope Constraint:** This firmware is **sensing-and-information only**. It does not control actuators that act against external objects. The Identification Sensor driver respects the hardware kill-switch state at all times. No override path exists in firmware.

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
- Radios: BLE 5.x (mandatory), UWB (optional)

---

## 3. Workspace Structure

The `firmware/` directory is a Cargo workspace. It shares code with the main `crates/` workspace where possible (protocol definitions, types), but compiles independently for embedded targets.

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
    │   └── identification_sensor.rs
    ├── logic/
    │   ├── mod.rs
    │   ├── body_frame_local.rs
    │   ├── presence_detection.rs
    │   └── power_management.rs
    └── ban_radio/
        ├── mod.rs
        └── protocol.rs
```

---

## 4. Node Types and Profiles

Each node type is a separate binary (`src/bin/*.rs`), compiled with features enabled for its specific sensor set.

### 4.1 Belt Node (`belt_node.rs`)

**Role:** Primary compute unit, network coordinator, fusion hub.

**Hardware:**
- MCU: High-performance (Cortex-M7 or equivalent Linux-capable SoM)
- Sensors: mmWave radar (long-range, downward-facing), IMU (torso reference), Environmental.
- Radio: BLE + optional Wi-Fi / Cellular for remote connectivity.

**Firmware Responsibilities:**
- Run full multi-modal fusion (`sentinel-fusion` crate, if MCU permits).
- Route BAN traffic between all nodes.
- Run PentaTrack (`sentinel-tracking` crate) for body-frame prediction.
- Expose API for companion app (phone, PC).

**Power Strategy:**
- Highest power budget (largest battery).
- Active fusion during movement; low-power presence monitoring otherwise.
- Must support continuous operation of compute loop + BAN hub duties simultaneously.

---

### 4.2 Pendant Node (`pendant_node.rs`)

**Role:** Upper-hemisphere sensing, identification hub.

**Hardware:**
- MCU: Mid-range (Cortex-M4F)
- Sensors: mmWave radar, IMU, Microphone array, Identification sensor (opt-in).

**Firmware Responsibilities:**
- Detect upper-body threats via radar.
- Full-echo acoustic profiling and direction-of-arrival estimation.
- **Identification Sensor Logic:**
  - Check `KILL_SWITCH_GPIO` at boot. Return `SensorError::KillSwitchEngaged` and halt if disabled.
  - Process identification frames locally; transmit only classification tags.
  - Never buffer or transmit raw frames.

**Power Strategy:**
- Event-triggered activation for ID sensor.
- Radar/IMU always-on at low duty cycle.

---

### 4.3 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

**Role:** Forearm-frame coverage, directional alerts.

**Hardware:**
- MCU: Low-power (Cortex-M4)
- Sensors: mmWave radar, IMU, Haptic actuator.

**Firmware Responsibilities:**
- Radar: Forearm hemisphere scanning.
- IMU: Arm motion, gesture detection, body-frame pose contribution.
- Haptic: Execute patterns received from belt controller `HapticCommand` messages.

**Power Strategy:**
- Aggressive sleep between polls.
- Haptic peak current management (back-EMF sensing for LRA resonance tracking).

---

### 4.4 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

**Role:** Ground-plane detection, gait analysis.

**Hardware:**
- MCU: Low-power
- Sensors: LiDAR/ToF (short-range), IMU, Haptic.

**Firmware Responsibilities:**
- ToF: Ground-level object detection and floor-clearance measurement.
- IMU: Gait pattern analysis (step detection, heel-strike, stumble precursor).
- Haptic: Directional alerts from below.

---

### 4.5 Eyewear Node (`eyewear_node.rs`)

**Role:** Forward field, high-speed event sensing, head-pose estimation.

**Hardware:**
- MCU: Low-latency variant (Cortex-M4/M33)
- Sensors: Event sensor, IMU.

**Firmware Responsibilities:**
- Event sensor processing (microsecond latency spike-stream).
- Spike-stream trajectory extraction (fast transient detection).
- Forward-collision warning logic.
- IMU: Head-pose estimation for body-frame separation of head vs. torso rotation.

**Power Strategy:**
- Event sensor is low-power but always active.
- IMU high-rate for head-motion stabilization.

---

## 5. Architecture Layers

### 5.1 HAL (Hardware Abstraction)

The firmware relies on `embedded-hal` traits for portability across MCU families.

```rust
// Example driver instantiation
use embedded_hal::spi::SpiDevice;
use embedded_hal::digital::OutputPin;

pub struct MmWaveDriver<SPI, CS> {
    spi: SPI,
    cs: CS,
}

impl<SPI, CS> MmWaveDriver<SPI, CS>
where
    SPI: SpiDevice,
    CS: OutputPin,
{
    pub fn new(spi: SPI, cs: CS) -> Self {
        Self { spi, cs }
    }
    // poll_presence(), poll_velocity(), configure_chirp() ...
}
```

### 5.2 Driver Layer (`src/drivers/`)

Each driver implements a generic interface, allowing hardware substitution without changing logic.

**`mmwave.rs`:**
- Implements `poll_presence()` and `poll_velocity()`.
- Handles chirp configuration (FMCW parameters) via SPI/UART.
- Micro-Doppler extraction for activity classification.

**`imu.rs`:**
- Polls at 200 Hz minimum.
- Runs local Madgwick filter (from `omni-sense-physics`).
- Outputs integrated orientation quaternions and gravity-compensated linear acceleration.

**`acoustic.rs`:**
- PDM/I2S read from microphone array.
- Runs on-MCU delay-and-sum beamforming (from `omni-sense-physics::acoustic`).
- Outputs direction-of-arrival and event-class metadata only.

**`lidar_tof.rs`:**
- UART/SPI interface to ToF module.
- Outputs distance readings and 8×8 zone maps.

**`event_sensor.rs`:**
- High-bandwidth event stream ingestion via DMA.
- Spatiotemporal clustering for motion event extraction.
- Outputs `MotionEvent` metadata, not raw pixel events.

**`haptic.rs`:**
- I2C driver for DRV2605L haptic controller.
- LRA resonance tracking via back-EMF feedback.
- Pre-defined pattern library (directional alert patterns encoded as waveform indices).

**`identification_sensor.rs`:**
- **Must** read `KILL_SWITCH_PIN` on initialization.
- Returns `IdentificationEvent` only; never raw frames.
- If kill-switch DISABLED: returns `Err(SensorError::KillSwitchEngaged)` immediately; does not proceed.

### 5.3 Logic Layer (`src/logic/`)

**`body_frame_local.rs`:**
- Integrates local IMU data using Madgwick/Mahony filter.
- Estimates node orientation relative to torso reference (requires calibration data from belt controller).
- Provides `stabilize_detection()` to convert sensor-frame detections to torso-aligned frame.

**`presence_detection.rs`:**
- Radar presence polling state machine.
- Hysteresis filter to suppress clutter and spurious motion.
- Outputs `PresenceEvent` with classification hint.

**`power_management.rs`:**
- Duty cycle control via RTOS task priorities.
- Sleep state entry/exit based on interrupt activity.
- Battery level monitoring via fuel gauge IC (I2C).
- Reports `BatteryStatus` to belt controller on change and on request.

---

## 6. BAN Protocol Stack (`src/ban_radio/`)

The Body Area Network protocol is optimized for:
- Low latency (sub-millisecond event alerts to haptic output).
- Minimal data transmission (processed metadata, not raw sensor data).
- Secure pairing (challenge-response at commissioning time).

**Message Types:**

```rust
pub enum SentinelBanMessage {
    NodeSensorData {
        timestamp_ms: u64,
        radar_metadata: RadarMetadata,   // Not raw chirp data
        imu_quaternion: [f32; 4],
        linear_accel: [f32; 3],
        node_id: NodeId,
    },
    HapticCommand {
        pattern: HapticPattern,
        duration_ms: u16,
        intensity: u8,                   // 0–100
    },
    IdentificationResult {
        classification: ClassificationTag,
        confidence: f32,
        node_id: NodeId,
    },
    GaitEvent {
        event_type: GaitEventType,       // Step, Stumble, Fall, Sprint, Standing
        confidence: f32,
    },
    AcousticEvent {
        event_class: String,
        direction_of_arrival: (f32, f32),
        confidence: f32,
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
}
```

---

## 7. Power Management Strategy

The firmware uses a layered sleep strategy. Each node's power budget is determined by its battery capacity and sensor duty cycle.

### 7.1 Sleep States

| State | CPU | Peripherals | Radio | Wake Source |
|---|---|---|---|---|
| **Active** | Full speed | All on | Connected | N/A |
| **Idle** | WFI halted | Sensors active | Connected | Timer, interrupt |
| **Standby** | Off (RAM retained) | Key sensors on | Advertising | PIR, radar, timer |
| **Deep Sleep** | Off | RTC only | Off | External interrupt only |

### 7.2 Node Lifecycle

1. **Boot & Init:** Peripherals up, check kill switches (identification nodes), read fuel gauge.
2. **Pairing/Commission:** Await pairing command from belt; exchange security credentials.
3. **Calibration:** Await `CalibrationCommand` from belt; stream high-rate IMU data for walk-through calibration.
4. **Normal Operation:**
   - Timer wakes node every `T_poll`.
   - Poll sensors per duty cycle configuration.
   - Transmit metadata to belt controller.
   - Enter Idle or Standby.
5. **Event Mode:** Detection triggers immediate priority wake and transmit.
6. **Low Battery:** Reduce sensor duty cycles; alert belt controller.

---

## 8. Build System

### `Cargo.toml` (Firmware Workspace)

```toml
[workspace]
resolver = "2"
members = [
    ".",
]

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

[profile.dev]
opt-level = 1
debug = true
lto = false

[profile.release]
opt-level = "z"
debug = false
lto = true
codegen-units = 1

[[bin]]
name = "pendant_node"
path = "src/bin/pendant_node.rs"

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
}
```

---

## 9. Identification Node Architecture

The Identification Sensor is on a physically isolated sub-circuit of the pendant node.

**Boot Sequence (mandatory — no bypass path):**
1. `main()` initializes GPIO and RTC only.
2. Reads `IDENTITY_KILL_SWITCH` GPIO pin.
3. If `LOW` (disabled):
   - Logs `KillSwitchEngaged` to BAN radio.
   - Loops forever — does NOT initialize camera, does NOT proceed.
4. If `HIGH` (enabled):
   - Initializes identification sensor driver.
   - Awaits `EnableIdentification` command from belt controller before capturing any data.
   - All frame capture → inference → result extraction occurs on-MCU.
   - Only `IdentificationResult` messages transmit over BAN.

**Absolute prohibitions (enforced architecturally, not by policy):**
- Never write raw frames to external flash.
- Never transmit raw frames over BAN or any other interface.
- Never buffer frames in RAM after inference completes.

---

## 10. Debug & Test

**On-Target Debug:**
- SWD (Serial Wire Debug) via 4-pin debug header.
- RTT (Real Time Transfer) for non-intrusive printf-style logs.
- UART console at 115200 baud for boot diagnostics.

**Host Simulation (`std` build):**
- `src/lib.rs` builds as `std` for host-side unit testing.
- Mock `embedded-hal` implementations provided for CI.
- Every driver has a `Simulated*` variant (mirrors OMNI-SENSE driver pattern).

---

## 11. Legal & Compliance

- **Sensing-Only:** This firmware does not control effectors. No actuator output path exists beyond the haptic motor (wearer-facing only).
- **Privacy:** Identification sensor is gated by hardware kill-switch. No firmware bypass.
- **Export Control:** No controlled sensing parameters. Simulation-only data paths.
- **Radio:** FCC Part 15 (US), CE RED (EU) compliance required for BLE radio usage. Ensure radio module is certified.
- **Battery:** Firmware must enforce voltage cutoffs (4.25 V overcharge, 3.0 V overdischarge). Hardware protections are primary; firmware is secondary enforcement layer.
