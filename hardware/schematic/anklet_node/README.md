# Anklet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Lower leg / above ankle — worn on both left and right ankles
**Primary Role:** Ground-plane sensing, gait analysis, lower-hemisphere coverage, haptic alerts, gait-phase timing synchronization
**Status:** Reference Design — Full Configuration Options
**License:** CERN-OHL-S v2

---

## 1. Connectivity Architecture

**The Anklet Node has NO WiFi and NO cellular connectivity.**

All anklet data transmits via Body-Area Network (BAN) to the belt node exclusively. The belt node is the sole external network gateway.

```
Anklet Sensors → BAN (BLE 5.x / UWB) → Belt Node → WiFi/Cellular → Companion App
```

The anklet never connects directly to the companion app, internet, or any external network.

### Why This Architecture

| Constraint | WiFi on Anklet | BLE on Anklet | UWB on Anklet |
|------------|----------------|---------------|---------------|
| Power | 300-1000+ mW | 5-15 mW | 50-150 mW |
| Heat | Significant | Minimal | Moderate |
| Antenna requirement | Large, body-blocked | Small PCB antenna | Moderate clearance |
| Form factor | Breaks jewelry scale | Preserves form | Compatible |
| Wearability | Poor | Excellent | Good |

**Conclusion:** WiFi is not viable for anklet-scale wearable devices. BLE is the primary BAN transport. UWB is optional for precision timing and burst data.

---

## 2. Purpose

The Anklet Node is the ground-level awareness and gait analysis layer. Its position on the lower leg provides:

- **Ground-plane sensing:** Detection of floor-level objects, obstacles, and approaching bodies from below
- **Gait analysis:** Primary input for step detection, heel-strike, cadence, stride asymmetry, and stumble precursor detection
- **Body-frame calibration:** High-motion profile during walking makes anklets the best calibration anchors for walk-through body-frame calibration
- **Haptic alerts:** Directional haptic for ground-level threats and rear/low directional cues
- **Gait-phase timing:** Precise timing of stance/swing/strike/push phases for body-frame fusion
- **Optional visual capture:** Downward or forward-downward camera for ground-level recording

---

## 3. Hardware Design Philosophy — Single PCB, Configuration Options

### Design Principle

The anklet node uses a **single PCB design** that supports all variants through component population and configuration options. This reduces SKU count while maintaining full capability range.

**Hardware variants become configuration options:**

| Previous Naming | Becomes |
|-----------------|---------|
| Variant A — Minimal | Configuration: Minimal sensor set |
| Variant B — Extended | Configuration: Extended sensor set with optional camera |

### PCB Design Includes Footprints For

| Component | Footprint | Populated When |
|-----------|-----------|----------------|
| ToF sensor | Yes | All configurations |
| LiDAR module | Yes | Extended configurations |
| mmWave radar | Yes | Extended configurations |
| Camera connector | Yes | When camera option selected |
| UWB module | Yes | When UWB option selected |
| Dual antenna | Yes | When antenna diversity option selected |
| Environmental sensor | Yes | Extended configurations |

**All options are user-configured at assembly time or can be modified post-purchase.**

---

## 4. Node Roles and Capabilities

### 4.1 Sensing Roles

| Role | Sensors Involved | Output |
|------|------------------|--------|
| Ground obstacle detection | ToF / LiDAR | Distance to obstacles at ankle height |
| Floor surface classification | IMU (impact analysis) + optional camera | Surface type (hard/soft/irregular) |
| Gait event detection | IMU | Step, heel-strike, toe-off events |
| Gait phase timing | IMU at 200 Hz | Current phase (stance/swing/strike/push) |
| Stumble precursor | IMU (asymmetric loading detection) | StumblePrecursor event |
| Approach from below | mmWave (optional) | Detection of low-approaching objects |
| Ground-level recording | Camera (optional) | Visual context of ground plane |

### 4.2 Haptic Alert Roles

Anklet haptics provide **lower-hemisphere directional cues**:

| Approach Direction | Haptic Location | Pattern |
|--------------------|-----------------|---------|
| Rear-left | Left anklet | Single pulse or directional pattern |
| Rear-right | Right anklet | Single pulse or directional pattern |
| Rear (center) | Both anklets | Simultaneous pulse |
| Ground obstacle | Affected anklet | Double pulse warning |
| Stumble precursor | Both anklets | Sustained buzz 300ms |

### 4.3 Calibration Role

Anklets are **primary calibration anchors** for walk-through body-frame calibration:

- High motion profile during walking
- Predictable leg swing arc provides strong geometric constraints
- IMU orientation data during walk establishes relative position to torso (belt IMU)
- Stride length estimation from paired anklet data validates calibration

---

## 5. Component Specifications

### 5.1 MCU Options

| MCU | Architecture | BLE | Power | Notes |
|-----|--------------|-----|-------|-------|
| Silicon Labs EFR32BG24 | Cortex-M33 | 5.3 | Ultra-low | Recommended for minimal config |
| Nordic nRF5340 | Dual Cortex-M33 | 5.3 | Low | Consistent firmware with other nodes |
| STM32L4 | Cortex-M4 | Via module | Ultra-low | Extended ADC for surface classification |

### 5.2 Short-Range Sensing

**ToF Sensors:**

| Sensor | Resolution | Range | Interface | Power | Notes |
|--------|------------|-------|-----------|-------|-------|
| STMicro VL53L5CX | 8×8 zones | 6 m | I2C | ~20 mW active | Primary choice |
| Terabee TeraRanger Evo Mini | Single point | 12 m | UART | ~15 mW | Lightweight alternative |

**Short-Range LiDAR (Extended Configurations):**

| Sensor | Range | Resolution | Interface | Power |
|--------|-------|------------|-----------|-------|
| TFMini-S | 12 m | 1 cm | UART | ~120 mW |
| Benewake TF-Luna | 8 m | 1 cm | UART/I2C | ~100 mW |
| Livox Mid-360 (research) | 40 m | Dense point cloud | Ethernet | 5-10 W (requires external power) |

### 5.3 mmWave Radar (Optional)

| Sensor | Frequency | Range | Interface | Power | Notes |
|--------|-----------|-------|-----------|-------|-------|
| Acconeer XR112 | 60 GHz | 3-5 m | SPI | 30-100 mW | Forward-facing, detects approaching objects |
| Infineon BGT60ATR24C | 60 GHz | 4 m | SPI | 50-150 mW | Custom antenna design possible |

**Antenna orientation:** Outward-facing (forward or lateral) from ankle position. Body absorption is significant — antenna must face away from the leg.

### 5.4 IMU — Critical Component

The anklet IMU is the **highest-dynamic-range sensor** in the SENTINEL-WEAR system.

| Requirement | Value | Reason |
|-------------|-------|--------|
| Accelerometer range | ±16 g minimum, ±32 g preferred | Heel-strike peaks at 5-10 g; sprinting higher |
| Gyroscope range | ±2000 °/s minimum | Leg swing during running |
| Sample rate | ≥ 200 Hz continuous | Gait phase resolution |
| Noise density | < 100 µg/√Hz | Step detection accuracy |

**Recommended IMUs:**

| IMU | Accelerometer | Gyroscope | Interface | Notes |
|-----|---------------|-----------|-----------|-------|
| Bosch BMI270 | ±16 g | ±2000 °/s | SPI/I2C | Consistent across all SENTINEL-WEAR nodes |
| TDK ICM-42688-P | ±32 g | ±4000 °/s | SPI | Smallest package, highest range |
| ST LSM6DSV | ±16 g | ±2000 °/s | SPI | Built-in ML for step detection (reduces MCU load) |

### 5.5 Camera Module (Optional)

| Sensor | Resolution | Power | Interface | Notes |
|--------|------------|-------|-----------|-------|
| HiMax HM01B0 | 320×240 (QVGA) | 2.1 mW | MIPI/DVP | Ultra-low power, ankle-appropriate |
| OmniVision OV2640 | 2 MP | 100-150 mW | DVP | Standard resolution, wide-angle option |
| Sony IMX219 | 8 MP | 300-500 mW | MIPI | Higher resolution, miniaturized fisheye available |

**Camera orientation options:**
- **Downward-facing:** Captures floor surface, ground obstacles, step edges
- **Forward-downward (30° from horizontal):** Captures approaching ground-level objects and forward floor area

**Data handling:** Fully user-configured per `sentinel-wear.toml`. Default: disabled. Options include metadata-only, local storage, or streaming via belt relay.

### 5.6 Haptic Actuator

| Actuator | Type | Size | Resonance | Driver |
|----------|------|------|-----------|--------|
| Precision Microdrives C10-100 | LRA | 10 mm | 150 Hz | DRV2605L (I2C) |
| AAC Technologies ALPS HF2-C3 | LRA | 8 mm (thin) | 235 Hz | DRV2605L |
| Precision Microdrives 307-103 | ERM | 8 mm | N/A | DRV2605L (simpler) |

**Driver:** TI DRV2605L
- I2C interface
- 123 built-in haptic patterns
- LRA resonance tracking via back-EMF
- Automatic braking for crisp pattern stops

### 5.7 Environmental Sensor (Extended Configurations)

| Sensor | Measurements | Interface | Power |
|--------|--------------|-----------|-------|
| Bosch BME688 | Temperature, humidity, pressure, VOC | I2C | ~3 mA active |
| Sensirion SEN55 | PM2.5, PM10, temp, humidity, VOC, NOx | I2C | ~5 mA active |

**Use case:** Surface classification context (wet floor, high humidity indicates slippery conditions). Not essential for basic gait detection.

---

## 6. Body-Area Network (BAN) Interface

### 6.1 BLE Radio — Primary Transport

**Present on ALL anklet configurations.**

| Specification | Value |
|---------------|-------|
| Standard | BLE 5.3 |
| Role | Peripheral (connects to belt as central) |
| Data carried | Gait events, detection metadata, health telemetry |
| Max sustained bandwidth | ~500 Kbps practical |
| Power (active) | 5-15 mW |
| Power (sleep) | < 0.5 mW |

### 6.2 UWB Radio — Optional Upgrade

**Recommended for anklets** due to gait-phase timing accuracy requirements.

| Specification | Value |
|---------------|-------|
| Module | Qorvo DW3000 class |
| Role | Precision time synchronization + ranging |
| Time sync accuracy | Sub-nanosecond |
| Ranging accuracy | < 10 cm |
| Bandwidth (data) | Up to 6.8 Mbps peak |
| Power (active) | 50-150 mW |
| Power (sleep) | < 1 mW |

**Why UWB is valuable for anklets:**

| Application | BLE Accuracy | UWB Accuracy | Impact |
|-------------|--------------|--------------|--------|
| Gait phase sync between left/right | ~10-30 ms | < 1 ms | Better stumble detection correlation |
| Step timing precision | ~10-30 ms | < 1 ms | More accurate cadence measurement |
| Stride length estimation | ~5-10 cm error | < 1 cm error | Better calibration |
| Body-frame reconstruction | Moderate | High | More accurate body model |

**UWB is NOT for external connectivity.** It communicates only with the belt node's UWB module. No external network access.

### 6.3 BAN Scheduling

Anklets are **Priority 2** (high) in the BLE scheduling hierarchy:

| Priority | Node | Reason |
|----------|------|--------|
| 1 | Belt IMU | Torso reference |
| **2** | **Anklets** | **Gait phase timing, stumble detection** |
| 3 | Pendant | Primary sensing |
| 4 | Bracelets | Secondary sensing |
| 5 | Eyewear | Optional |

**Scheduled slot (30 ms connection interval):**
```
Slot 5-7 ms:   Belt → Anklet L (sync, command)
Slot 7-10 ms:  Anklet L → Belt (gait event)
Slot 10-12 ms: Belt → Anklet R (sync)
Slot 12-15 ms: Anklet R → Belt (gait event)
```

**During walking context:** Anklet priority can be boosted (slot intervals reduced) for more frequent gait updates.

### 6.4 Antenna Diversity Option

Anklet PCB can support dual BLE antennas for diversity:

| Configuration | Description |
|---------------|-------------|
| Single antenna | Standard |
| Dual antenna diversity | Front and rear antennas, radio selects best |

**Why antenna diversity matters for anklets:**
- Ankle position is highly variable (leg swing during walking)
- Body shadowing from the other leg during gait
- Ground-level obstruction (standing in tall grass, obstacles)
- Diversity improves reliability during movement

**Configuration:**
```toml
[nodes.anklet_left.antenna_diversity]
enabled = false  # Set true if dual-antenna PCB populated
selection_mode = "rssi"
switch_threshold_db = 5
```

---

## 7. Data Contracts — Anklet to Belt

### 7.1 Standard Gait Event Message

```rust
/// Gait event transmitted from anklet to belt
pub struct GaitEvent {
    /// Event type
    pub event_type: GaitEventType,
    /// Current gait phase
    pub phase: GaitPhase,
    /// Confidence in classification (0.0 - 1.0)
    pub confidence: f32,
    /// Peak acceleration magnitude during heel-strike (g)
    pub impact_g: Option<u8>,
    /// Timestamp (from synchronized clock)
    pub timestamp_ns: u64,
}

pub enum GaitEventType {
    HeelStrike,
    ToeOff,
    StepComplete,
    StumblePrecursor,
    IrregularStep,
}

pub enum GaitPhase {
    Stance,     // Foot on ground
    Swing,      // Foot in air
    Strike,     // Heel-strike occurring
    Push,       // Push-off phase
}
```

### 7.2 Detection Event Message

```rust
/// Detection from anklet sensors
pub struct AnkletDetection {
    /// Detection class
    pub class: DetectionClass,
    /// Range to detected object (meters)
    pub range_m: f32,
    /// Bearing relative to anklet forward (radians)
    pub bearing_rad: f32,
    /// Detection confidence
    pub confidence: f32,
    /// Timestamp
    pub timestamp_ns: u64,
}

pub enum DetectionClass {
    GroundObstacle,
    FloorEdge,
    ApproachingLowObject,
    Unknown,
}
```

### 7.3 Battery and Health Message

```rust
/// Battery and health telemetry (sent every 10 seconds)
pub struct NodeHealth {
    /// Battery percentage (0-100)
    pub battery_percent: u8,
    /// Battery voltage (mV)
    pub voltage_mv: u16,
    /// Charging state
    pub charging: bool,
    /// Sensor health status
    pub sensor_health: Vec<SensorHealthStatus>,
    /// Temperature (°C)
    pub temperature_c: i8,
}

pub enum SensorHealthStatus {
    Ok,
    Degraded,
    Failed,
}
```

### 7.4 On-Node Processing

**Anklets perform on-node processing to reduce BAN bandwidth:**

| Processing | On-Node | Transmitted |
|------------|---------|-------------|
| IMU raw samples (200 Hz) | ✅ Gait event detection | Gait events only (once per step) |
| ToF distance readings | ✅ Obstacle detection | Detection events only |
| mmWave raw data | ✅ Presence detection | Detection events only |
| Camera frames (if present) | ✅ Optional compression | Compressed stream or clips |
| Stumble precursor | ✅ Full analysis on-node | StumblePrecursor event |

**Raw IMU burst transmission:** Only during anomaly detection (stumble precursor) or calibration mode. Normal operation sends processed events, not raw samples.

---

## 8. Power Architecture

### 8.1 Power Domains

| Rail | Voltage | Source | Usage |
|------|----------|--------|-------|
| VBAT | 3.7-4.2 V | LiPo battery | Direct to haptic driver |
| VCC_3V3 | 3.3 V | LDO/buck from VBAT | MCU, sensors, BLE radio |
| VCC_1V8 | 1.8 V | LDO | Low-power IMU mode |
| V_CAM | 2.8/3.3 V | Switched | Camera module (optional) |
| V_UWB | 3.3 V | Switched | UWB module (optional) |

### 8.2 Power Budget

| Mode | Active Components | Power Draw | Runtime (300 mAh) | Runtime (500 mAh) |
|------|-------------------|------------|-------------------|-------------------|
| Sleep | BLE advertising only | < 1 mW | ~1000+ hours | — |
| Idle | BLE connected, sensors idle | 5-10 mW | ~100+ hours | — |
| Active (walking) | BLE + IMU + ToF | 30-60 mW | ~20-30 hours | ~30-50 hours |
| Active (extended + camera) | All + camera | 100-200 mW | ~5-10 hours | ~10-15 hours |
| UWB active | BLE + UWB | 60-100 mW | ~12-20 hours | ~20-30 hours |

### 8.3 Battery Configuration Options

| Battery | Capacity | Weight | Thickness | Runtime (Active) |
|---------|----------|--------|-----------|------------------|
| Small | 200 mAh | ~5 g | 3 mm | 15-20 hours |
| Standard | 300 mAh | ~7 g | 4 mm | 20-30 hours |
| Extended | 500 mAh | ~12 g | 5 mm | 30-50 hours |

**Configuration:**
```toml
[nodes.anklet_left.battery]
capacity_mah = 300
low_power_threshold_percent = 20
critical_threshold_percent = 10
```

### 8.4 Charging

| Method | Interface | Power | Notes |
|--------|-----------|-------|-------|
| Qi wireless | Coil in band | 5 W | Most convenient for anklet |
| Magnetic pogo | Dock contact | 5 W | Faster charging, requires dock |
| USB-C | USB connector | 5-20 W | Development/debug only |

---

## 9. Thermal Considerations

### 9.1 Heat Sources

| Component | Power | Duty Cycle | Heat Contribution |
|-----------|-------|------------|-------------------|
| MCU | 20-50 mW | 100% | Moderate |
| BLE radio | 10-15 mW | 10% | Low |
| IMU | 1-3 mW | 100% | Minimal |
| ToF | 20 mW | 50% | Low |
| Camera (optional) | 100-300 mW | 10% | Moderate |
| UWB (optional) | 50-100 mW | 20% | Moderate |

### 9.2 Thermal Management

**Total worst-case power (extended config with camera):** ~200 mW

**Enclosure design:**
- Ankle position benefits from air circulation during walking
- Silicone band provides thermal insulation from skin
- Maximum sustained enclosure temperature: < 40°C

**Firmware thermal monitoring:**
```toml
[thermal]
warning_threshold_c = 40
throttle_threshold_c = 45
shutdown_threshold_c = 50
throttle_action = "reduce_camera_fps"  # "reduce_camera_fps" | "disable_camera"
```

---

## 10. Gait Analysis Integration

### 10.1 Processing Pipeline

```
IMU (200 Hz continuous)
    │
    ├─── Accelerometer data
    │        │
    │        ├─── Bandpass filter (1-10 Hz) → Step detection
    │        ├─── Peak detection → Heel-strike timestamp
    │        └─── Impact analysis → Surface classification
    │
    ├─── Gyroscope data
    │        │
    │        ├─── Integration → Leg angle
    │        └─── Angular rate → Swing velocity
    │
    └─── Sensor fusion (accel + gyro)
             │
             ├─── Gait phase classifier
             └─── Stumble precursor detection
                     │
                     └─── Anomaly score > threshold → StumblePrecursor event
```

### 10.2 Step Detection Algorithm (On-Node)

```rust
/// On-node step detection (simplified)
pub fn detect_step(imu_sample: &ImuSample, state: &mut GaitState) -> Option<GaitEvent> {
    // Vertical acceleration after gravity removal
    let vertical_accel = imu_sample.accel_z - state.gravity_estimate;
    
    // Bandpass filter (1-10 Hz) to isolate step frequency
    let filtered = state.bandpass_filter.update(vertical_accel);
    
    // Peak detection for heel-strike
    if state.peak_detector.is_peak(filtered) {
        let now = get_timestamp_ns();
        let step_interval = now - state.last_heel_strike_ns;
        
        // Validate step interval (200 ms - 2 seconds typical)
        if step_interval > 200_000_000 && step_interval < 2_000_000_000 {
            state.last_heel_strike_ns = now;
            state.step_count += 1;
            
            // Calculate impact magnitude
            let impact_g = filtered.abs() / 9.81;
            
            return Some(GaitEvent {
                event_type: GaitEventType::HeelStrike,
                phase: GaitPhase::Strike,
                confidence: 0.95,
                impact_g: Some(impact_g as u8),
                timestamp_ns: now,
            });
        }
    }
    
    None
}
```

### 10.3 Stumble Precursor Detection

```rust
/// Stumble precursor detection
pub fn detect_stumble_precursor(
    left_anklet: &AnkletState,
    right_anklet: &AnkletState,
) -> Option<StumblePrecursor> {
    // Asymmetric loading: one foot has high impact, other has irregular timing
    let impact_asymmetry = (left_anklet.last_impact_g - right_anklet.last_impact_g).abs();
    let timing_asymmetry = (left_anklet.step_interval - right_anklet.step_interval).abs();
    
    // Balance recovery: high-frequency micro-accelerations after irregular step
    let balance_instability = left_anklet.high_freq_energy + right_anklet.high_freq_energy;
    
    // Combined score
    let stumble_score = 
        impact_asymmetry * 0.4 +
        (timing_asymmetry / 100_000_000.0) * 0.3 +  // Normalize by expected step interval
        balance_instability * 0.3;
    
    if stumble_score > STUMBLE_THRESHOLD {
        return Some(StumblePrecursor {
            score: stumble_score,
            confidence: 0.85,
            timestamp_ns: get_timestamp_ns(),
        });
    }
    
    None
}
```

---

## 11. Interface Summary

### 11.1 Standard Interfaces (All Configurations)

| Interface | Signal Names | Description |
|-----------|--------------|-------------|
| ToF | `TOF_SDA`, `TOF_SCL`, `TOF_INT` | I2C or UART |
| IMU | `IMU_CS`, `IMU_INT`, `IMU_CLK`, `IMU_MISO`, `IMU_MOSI` | SPI |
| Haptic | `HAPTIC_SDA`, `HAPTIC_SCL`, `HAPTIC_EN` | I2C + GPIO |
| BAN Radio | Integrated in MCU | BLE 5.3 |
| Charging | `CHRG_IN+`, `CHRG_IN-` | Qi coil or pogo pins |
| Debug | `SWD_CLK`, `SWD_DATA`, `UART_TX`, `UART_RX` | 10-pin ARM Debug |

### 11.2 Optional Interfaces (Extended Configurations)

| Interface | Signal Names | Description |
|-----------|--------------|-------------|
| mmWave radar | `MMWAVE_CS`, `MMWAVE_CLK`, `MMWAVE_MISO`, `MMWAVE_MOSI`, `MMWAVE_IRQ` | SPI |
| LiDAR | `LIDAR_TX`, `LIDAR_RX`, `LIDAR_TRIG` | UART |
| Camera | `CAM_MIPI_CLK+/-`, `CAM_MIPI_D0+/-`, `CAM_PWDN` | MIPI CSI-2 |
| Camera (DVP) | `CAM_D[0:7]`, `CAM_PCLK`, `CAM_HSYNC`, `CAM_VSYNC` | Parallel |
| UWB | `UWB_SPI_*`, `UWB_IRQ`, `UWB_RST` | SPI to DW3000 |
| Environmental | `ENV_SDA`, `ENV_SCL` | I2C |

### 11.3 Antenna Diversity (Optional)

| Signal | Description |
|--------|-------------|
| `ANT_SEL_1` | Antenna select control 1 |
| `ANT_SEL_2` | Antenna select control 2 |
| `ANT_DIV_EN` | Diversity enable |

---

## 12. Mechanical Design

### 12.1 Placement

**Position:** Lower leg, above the ankle bone (malleolus). Can be medial (inside) or lateral (outside) — calibration handles both. Left and right anklets are hardware-identical; firmware handles coordinate convention.

**Orientation:** Sensor apertures face outward (forward or lateral) from the leg for optimal sensing.

### 12.2 Enclosure Requirements

| Requirement | Specification |
|-------------|---------------|
| Water resistance | IP67 mandatory (sweat, rain, mud, submersion) |
| Impact resistance | Survive 10 g heel-strike, walking into furniture |
| Skin contact | EN 1811 nickel release compliance |
| Band material | Medical-grade silicone, titanium, or surgical steel |
| Weight target | < 70 g (minimal), < 100 g (extended) |
| Thickness | < 15 mm profile preferred |

### 12.3 Form Factor Options

| Form | Description | Weight |
|------|-------------|--------|
| Slim band | Silicone strap with small central module | 40-60 g |
| Sport band | Wider strap, larger central module | 60-90 g |
| Premium | Titanium or surgical steel clasp | 70-100 g |

### 12.4 Sensor Apertures

| Sensor | Aperture Material | Notes |
|--------|-------------------|-------|
| ToF / LiDAR | Optical-grade polycarbonate | AR coating improves range |
| mmWave | RF-transparent (ABS, polycarbonate) | No metal in aperture zone |
| Camera | Optical-grade polycarbonate | AR coating, optional IR filter |
| Microphone (for surface classification) | Acoustic mesh / PTFE membrane | Waterproof, breathable |

---

## 13. Left vs. Right Handling

### 13.1 Hardware

Left and right anklets are **identical PCB assemblies**. No hardware differences.

### 13.2 Firmware

Two firmware binaries handle coordinate conventions:

| Binary | Purpose |
|--------|---------|
| `anklet_left.rs` | Assumes left-leg coordinate convention |
| `anklet_right.rs` | Assumes right-leg coordinate convention |

**Configuration:**
```toml
[nodes.anklet_left]
firmware = "anklet_left"
side = "left"

[nodes.anklet_right]
firmware = "anklet_right"
side = "right"
```

### 13.3 Coordinate Convention

Body-frame coordinate system (belt node reference):

| Direction | Positive Axis |
|-----------|---------------|
| Forward | +Y |
| Right | +X |
| Up | +Z |

**Anklet-specific:**
- Left anklet: positioned at (-X_anklet, Y_anklet, Z_anklet) relative to torso
- Right anklet: positioned at (+X_anklet, Y_anklet, Z_anklet) relative to torso

Firmware transforms local sensor readings to body-frame coordinates before transmission to belt.

---

## 14. Calibration Role

### 14.1 Walk-Through Calibration

Anklets are **primary calibration anchors** during walk-through body-frame calibration:

1. User walks normally for 30-60 seconds
2. Anklet IMUs record motion profiles
3. Belt controller trilaterates anklet positions from motion correlation
4. System estimates stride length, leg geometry, and anklet mounting position

### 14.2 Calibration Data Provided

| Data | Source | Purpose |
|------|--------|---------|
| Step timing | IMU heel-strike detection | Gait cadence baseline |
| Leg swing arc | IMU orientation during walk | Leg geometry estimation |
| Stride length | Paired anklet correlation | Distance calibration |
| Mounting position | Motion profile analysis | Body-frame position estimate |

### 14.3 Continuous Calibration Refinement

During normal use, the system continuously refines calibration:

- Each step provides new timing data
- Long-term gait patterns refine position estimates
- Calibration confidence improves with wear time

---

## 15. Configuration File Reference

```toml
# Anklet node configuration

[nodes.anklet_left]
enabled = true
firmware = "anklet_left"
side = "left"

[nodes.anklet_left.sensors]
# ToF always present
tof_enabled = true
tof_model = "VL53L5CX"

# LiDAR (extended)
lidar_enabled = false
lidar_model = "TFMini-S"

# mmWave (extended)
mmwave_enabled = false
mmwave_model = "XR112"

# IMU
imu_model = "BMI270"
imu_sample_rate_hz = 200

# Camera (extended)
camera_enabled = false
camera_model = "HM01B0"
camera_orientation = "downward"  # "downward" | "forward_downward"

# Environmental
environmental_enabled = false

# UWB
uwb_enabled = false
uwb_role = "timing_and_ranging"

[nodes.anklet_left.haptic]
enabled = true
default_intensity = 70  # 0-100
patterns = "standard"

[nodes.anklet_left.antenna_diversity]
enabled = false

[nodes.anklet_left.battery]
capacity_mah = 300
low_power_threshold_percent = 20
critical_threshold_percent = 10

[nodes.anklet_left.gait]
step_detection_enabled = true
stumble_detection_enabled = true
stumble_sensitivity = 0.7  # 0.0 - 1.0
impact_threshold_g = 5

# Mirror configuration for right anklet
[nodes.anklet_right]
enabled = true
firmware = "anklet_right"
side = "right"
# ... (same sensor configuration as left)
```

---

## 16. Testing Strategy

### 16.1 Electrical Tests

| Test | Pass Criteria |
|------|----------------|
| Power rails | Within ±5% of nominal |
| MCU boot | CRC verified, firmware loads |
| Sensor enumeration | All I2C/SPI sensors respond |
| BLE connection | Connects to golden belt node within 200 ms |
| UWB connection (if populated) | Ranging accuracy < 10 cm |

### 16.2 Functional Tests

| Test | Pass Criteria |
|------|----------------|
| ToF ranging | ±5 cm accuracy at 1 m |
| IMU gravity vector | Reads 9.81 ± 0.2 m/s² at rest |
| Step detection | 10 simulated steps → 10 step events |
| Heel-strike impact | 5 g impulse detected |
| Stumble precursor | Irregular step triggers StumblePrecursor event |
| Haptic activation | LRA resonates at 150 ± 15 Hz |

### 16.3 Environmental Tests

| Test | Pass Criteria |
|------|----------------|
| IP67 (1 m, 30 min) | Powers on, all sensors function |
| Thermal | Enclosure < 40°C after 1 hour active |
| Sweat exposure | No corrosion, continued operation |
| Impact (10 g) | No damage, continued operation |

### 16.4 Left/Right Configuration Test

| Test | Pass Criteria |
|------|----------------|
| Left firmware | Reports position in negative X hemisphere |
| Right firmware | Reports position in positive X hemisphere |
| Coordinate transform | Detections correctly mapped to body frame |

---

## 17. Directory Structure

```
anklet_node/
├── pcb_design/
│   ├── anklet_main.kicad_pro        # Single PCB design for all configs
│   ├── anklet_main.kicad_sch
│   ├── anklet_main.kicad_pcb
│   └── gerbers/
│       ├── anklet_main-F_Cu.gbr
│       ├── anklet_main-B_Cu.gbr
│       └── ...
├── bom/
│   ├── anklet_minimal_bom.csv
│   └── anklet_extended_bom.csv
├── configuration/
│   ├── minimal.toml
│   ├── standard.toml
│   └── extended.toml
├── test_fixtures/
│   └── gait_simulation_rig.step
└── README.md
```

---

## 18. Summary

The Anklet Node is the ground-level sensing and gait analysis layer of SENTINEL-WEAR:

- **Single PCB design** supports all configurations through component population
- **BLE-only or BLE+UWB** BAN connectivity to belt node
- **No WiFi, no cellular** — external connectivity only via belt
- **High-dynamic-range IMU** for gait analysis at 200 Hz
- **Priority-2 BLE scheduling** for gait timing precision
- **Antenna diversity option** for movement reliability
- **On-node gait processing** reduces BAN bandwidth
- **IP67 enclosure** for sweat, rain, mud exposure
- **Left/right firmware variants** for coordinate convention

**Primary role:** Gait analysis, ground-plane sensing, stumble precursor detection, body-frame calibration anchor.

---

**End of Anklet Node Hardware Specification**
