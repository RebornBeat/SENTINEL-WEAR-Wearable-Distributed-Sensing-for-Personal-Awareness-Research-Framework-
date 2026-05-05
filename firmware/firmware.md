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

**Scope Constraint:** This firmware is **sensing-and-information only**. It does not control actuators that act against external objects. The Identification Sensor driver respects the hardware kill-switch state.

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
- MCU: High-performance (Cortex-M7 or equivalent)
- Sensors: mmWave radar (long-range), IMU (high-rate), Environmental.
- Radio: BLE + optional Wi-Fi / Cellular for remote connectivity.

**Firmware Responsibilities:**
- Run full multi-modal fusion (`sentinel-fusion` crate, if MCU permits).
- Route BAN traffic.
- Run PentaTrack (`sentinel-tracking` crate) for body-frame prediction.
- Expose API for companion app.

**Power Strategy:**
- Highest power budget (largest battery).
- Active fusion during movement; low-power presence monitoring otherwise.

---

### 4.2 Pendant Node (`pendant_node.rs`)

**Role:** Upper-hemisphere sensing, identification hub.

**Hardware:**
- MCU: Mid-range (Cortex-M4F)
- Sensors: mmWave radar, IMU, Microphone array, Identification sensor (opt-in).

**Firmware Responsibilities:**
- Detect upper-body threats.
- Full-echo acoustic profiling.
- **Identification Sensor Logic:**
  - Check `KILL_SWITCH_GPIO` at boot. Halt if disabled.
  - Process frames locally; transmit only classification tags.

**Power Strategy:**
- Event-triggered activation for ID sensor.
- Radar/IMU always-on (low duty cycle).

---

### 4.3 Bracelet Nodes (`bracelet_left.rs`, `bracelet_right.rs`)

**Role:** Forearm-frame coverage, directional alerts.

**Hardware:**
- MCU: Low-power (Cortex-M4)
- Sensors: mmWave radar, IMU, Haptic actuator.

**Firmware Responsibilities:**
- Radar: Forearm hemisphere scanning.
- IMU: Arm motion gesture detection.
- Haptic: Execute patterns from belt controller commands.

**Power Strategy:**
- Aggressive sleep between polls.
- Haptic peak current management.

---

### 4.4 Anklet Nodes (`anklet_left.rs`, `anklet_right.rs`)

**Role:** Ground-plane detection, gait analysis.

**Hardware:**
- MCU: Low-power
- Sensors: LiDAR/ToF (short-range), IMU, Haptic.

**Firmware Responsibilities:**
- ToF: Ground-level object detection.
- IMU: Gait pattern analysis (step detection, stumble precursor).
- Haptic: Directional alerts.

---

### 4.5 Eyewear Node (`eyewear_node.rs`)

**Role:** Forward field, high-speed event sensing.

**Hardware:**
- MCU: Low-latency variant
- Sensors: Event sensor, IMU.

**Firmware Responsibilities:**
- Event sensor processing (microsecond latency).
- Spike-stream trajectory extraction.
- Forward-collision warning logic.

**Power Strategy:**
- Event sensor is low-power but always active.
- IMU high-rate for head-motion stabilization.

---

## 5. Architecture Layers

### 5.1 HAL (Hardware Abstraction)

The firmware relies on `embedded-hal` traits for portability.

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
    // ... implementation
}
```

### 5.2 Driver Layer (`src/drivers/`)

Each driver implements a generic interface, allowing hardware substitution without changing logic.

**`mmwave.rs`**:
- Implements `poll_presence()` and `poll_velocity()`.
- Handles chirp configuration (FMCW parameters).

**`imu.rs`**:
- Polls at 200 Hz.
- Runs local Madgwick filter (from `omni-sense-physics`).
- Outputs integrated orientation quaternions.

**`identification_sensor.rs`**:
- **Must** read `KILL_SWITCH_PIN` on initialization.
- Returns `IdentificationEvent`, never raw frames.
- Encrypted storage of temporary embeddings (if local storage is present).

### 5.3 Logic Layer (`src/logic/`)

**`body_frame_local.rs`**:
- Integrates local IMU data.
- Estimates node orientation relative to torso (requires calibration data).

**`presence_detection.rs`**:
- Radar presence polling.
- Filtering for motion vs static clutter.

**`power_management.rs`**:
- Duty cycle control.
- Sleep state entry/exit.
- Battery level monitoring.

---

## 6. BAN Protocol Stack (`src/ban_radio/`)

The Body Area Network protocol is optimized for:
- Low latency (sub-millisecond event alerts).
- Minimal data transmission (metadata only).
- Secure pairing.

**Message Types:**
```rust
enum SentinelBanMessage {
    NodeSensorData {
        timestamp_ms: u64,
        radar_data: RadarMetadata, // Not raw chirp data
        imu_quat: [f32; 4],
        node_id: NodeId,
    },
    HapticCommand {
        pattern: HapticPattern,
        duration_ms: u16,
    },
    IdentificationResult {
        classification: ClassificationTag,
        confidence: f32,
    },
    GaitEvent {
        event_type: GaitEventType, // Stumble, Fall, Sprint
    },
    BatteryStatus {
        percent: u8,
        voltage_mv: u16,
    },
}
```

---

## 7. Power Management Strategy

The firmware uses a layered sleep strategy.

### 7.1 Sleep States
- **Active:** All peripherals on.
- **Idle:** CPU halted, peripherals active.
- **Standby:** RAM retained, radio off, sensor polling disabled.

### 7.2 Node Lifecycle
1. **Boot & Init:** Peripherals up, check kill switches.
2. **Calibration:** Await `CalibrationCommand` from belt.
3. **Normal Operation:**
   - Timer wakes node every `T_poll`.
   - Poll sensors.
   - Transmit metadata.
   - Enter Idle.
4. **Event Mode:** Detection triggers immediate wake and transmit.

---

## 8. Build System

### `Cargo.toml` (Firmware Workspace)

```toml
[workspace]
resolver = "2"
members = [
    ".", # src/lib.rs
    "src/bin/pendant_node",
    "src/bin/belt_node",
    # ... other binaries
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
# Drivers
mmwave-driver = { path = "src/drivers/mmwave" }
# ... other deps

[profile.dev]
opt-level = 1
debug = true
lto = false

[profile.release]
opt-level = "z"
debug = false
lto = true
```

### `build.rs`

```rust
fn main() {
    println!("cargo:rustc-link-arg=Tlink.x");
    println!("cargo:rerun-if-changed=link.x");
}
```

---

## 9. Identification Node Architecture

The Identification Sensor is a separate "core" or isolated partition on the pendant node.

**Boot Sequence:**
1. `main()` initializes GPIO.
2. Reads `IDENTITY_KILL_SWITCH`.
3. If `LOW` (disabled):
   - Log disabled state.
   - Loop forever (do not init camera).
4. If `HIGH` (enabled):
   - Init Identification Sensor.
   - Await `EnableIdentification` command from belt.

**Data Flow:**
- Sensor Frame → Local Compute → `IdentificationEvent` → BAN Transmit.

---

## 10. Debug & Test

**On-Target Debug:**
- SWD (Serial Wire Debug) via debug header.
- RTT (Real Time Transfer) for logs.

**Host Simulation:**
- Use `std` build of `src/lib.rs` for unit tests.
- Mock `embedded-hal` implementations for CI testing.

---

## 11. Legal & Compliance

- **Sensing-Only:** This firmware does not control effectors.
- **Privacy:** Identification sensor is gated by hardware kill-switch.
- **Export Control:** No controlled parameters.
- **Radio:** FCC Part 15 (US), CE RED (EU) compliance required for radio usage.

---
