# Doppler Radar Configuration for Extreme Velocity Detection

**Project:** SENTINEL-WEAR
**Domain:** High-speed object detection and threat characterization
**Status:** Production-Capable Architecture
**Implementation:** `crates/sentinel-extreme-velocity/`, `firmware/src/drivers/mmwave.rs`

---

## 1. Purpose

This document specifies the Doppler radar configuration architecture for detecting high-velocity objects (projectiles, debris, fragments) within the SENTINEL-WEAR body-frame awareness field. Unlike standard automotive or industrial radar profiles designed for human-scale velocities, extreme velocity detection requires extended Doppler range, high update rates, and microsecond-latency signal processing.

**Key thesis:** Standard mmWave radar profiles are optimized for human and vehicle motion (velocities up to ~18 m/s). Projectile detection requires velocities up to 850+ m/s, requiring custom chirp configurations that trade range resolution for velocity range.

---

## 2. Physics of High-Velocity Detection

### 2.1 The Detection Challenge

| Projectile Type | Typical Velocity | Time to Traverse 3 m | Time to Traverse 1 m |
|-----------------|------------------|----------------------|----------------------|
| Handgun (9mm) | 300-400 m/s | 7.5-10 ms | 2.5-3.3 ms |
| Handgun (.45 ACP) | 250-300 m/s | 10-12 ms | 3.3-4 ms |
| Rifle (5.56 NATO) | 900-950 m/s | 3.2-3.3 ms | 1.0-1.1 ms |
| Rifle (7.62 NATO) | 800-850 m/s | 3.5-3.8 ms | 1.1-1.3 ms |
| High-velocity rifle | 1000+ m/s | < 3 ms | < 1 ms |

**The detection window is 2.5-12 milliseconds from entry to potential impact.**

### 2.2 Doppler Radar Fundamentals

**Doppler shift equation:**

$$\Delta f = \frac{2 \cdot v \cdot f_c}{c}$$

Where:
- $\Delta f$ = Doppler frequency shift (Hz)
- $v$ = radial velocity of target (m/s)
- $f_c$ = radar carrier frequency (Hz)
- $c$ = speed of light (3×10⁸ m/s)

**For 60 GHz radar:**

| Target Velocity | Doppler Shift |
|-----------------|---------------|
| 1 m/s (walking) | 400 Hz |
| 10 m/s (running) | 4 kHz |
| 100 m/s (slow projectile) | 40 kHz |
| 350 m/s (handgun) | 140 kHz |
| 850 m/s (rifle) | 340 kHz |
| 1000 m/s (high-velocity) | 400 kHz |

### 2.3 Standard vs Extended Doppler Range

**Standard automotive radar profile:**

| Parameter | Typical Value | Reason |
|-----------|---------------|--------|
| Max Doppler velocity | ±15-20 m/s | Human/vehicle speeds |
| Chirp duration | 20-40 µs | Good range resolution |
| Chirp bandwidth | 2-4 GHz | Range resolution ~10 cm |
| Update rate | 50-200 Hz | Sufficient for driving |

**Extended Doppler profile for projectile detection:**

| Parameter | Value | Tradeoff |
|-----------|-------|----------|
| Max Doppler velocity | ±300-500 m/s | Reduced range resolution |
| Chirp duration | 5-10 µs (shorter) | Velocity resolution degraded |
| Chirp bandwidth | 500 MHz - 1 GHz (narrower) | Range resolution ~30-60 cm |
| Update rate | 10,000+ Hz | Required for fast transients |

---

## 3. mmWave Module Capabilities

### 3.1 TI IWR6843 Family

**Overview:** 60-64 GHz FMCW radar, widely used in automotive and industrial applications.

**Standard configuration:**
- Max Doppler velocity: ~±18 m/s (standard chirp)
- Range resolution: ~4 cm (4 GHz bandwidth)
- Update rate: 10-200 Hz typical

**Extended Doppler configuration:**

The IWR6843 can be configured for extended velocity range through custom chirp parameters:

| Parameter | Standard | Extended Doppler |
|-----------|----------|------------------|
| Chirp bandwidth | 4 GHz | 500 MHz - 1 GHz |
| Chirp duration | 30 µs | 8-12 µs |
| Max velocity | ±18 m/s | ±300-400 m/s |
| Velocity resolution | 0.1 m/s | 2-5 m/s |
| Range resolution | 4 cm | 30-60 cm |
| Max range | 80 m | 20-40 m |

**Implementation:**

```c
// TI mmWave SDK chirp configuration for extended Doppler
// This is pseudo-code showing the parameter relationships

chirpConfig = {
    .profileId = 1,
    .chirpStartIndex = 0,
    .chirpEndIndex = 127,
    
    // Reduced bandwidth for extended Doppler
    .freqStartConst = 60e9,           // 60 GHz start
    .freqSlopeConst = 10e12,          // 10 MHz/µs slope (vs 40+ MHz/µs standard)
    
    // Shorter chirp duration
    .txStartTime = 1e-6,              // 1 µs idle time
    .rampEndTime = 10e-6,             // 10 µs ramp
    
    // High update rate
    .numLoops = 1,                    // Single chirp per frame
    .framePeriodicity = 100e-6,       // 100 µs = 10 kHz update rate
};

// Result: ±300 m/s Doppler range, 10 kHz update rate, ~50 cm range resolution
```

**Tradeoff summary:**

| Capability | Standard | Extended Doppler | Impact |
|------------|----------|------------------|--------|
| Velocity range | ±18 m/s | ±300 m/s | ✅ Projectile detection possible |
| Range resolution | 4 cm | 50 cm | ⚠️ Position less precise |
| Max range | 80 m | 30 m | ⚠️ Reduced for distant targets |
| Update rate | 50 Hz | 10,000 Hz | ✅ Millisecond detection |

**For SENTINEL-WEAR, the tradeoff is acceptable:**
- We care about detection latency, not precise ranging
- Engagement distances are < 10 m (typically 3-5 m)
- 50 cm range resolution is sufficient for "threat approaching" vs "threat passed"

### 3.2 Infineon BGT60ATR24C

**Overview:** 60 GHz MMIC radar, SPI interface, compact form factor.

**Standard configuration:**
- Max Doppler velocity: Configurable
- Chirp flexibility: High (fully programmable)
- Update rate: Up to 100 kHz raw data rate

**Extended Doppler capability:**
- Requires research to determine optimal chirp parameters
- SPI interface allows real-time chirp reconfiguration
- Suitable for on-node processing (MCU handles Doppler FFT)

**Research needed:** Characterize extended Doppler performance for this module.

### 3.3 Acconeer XR112

**Overview:** 60 GHz pulse radar, ultra-low power, SPI interface.

**Characteristics:**
- Different operating principle (pulse radar, not FMCW)
- Excellent for low-power presence detection
- Velocity detection: Limited compared to FMCW

**Suitability for extreme velocity:**
- Unknown; pulse radar may have advantages for single-shot velocity measurement
- Requires research

### 3.4 Module Selection Recommendations

| Module | Extended Doppler Support | Research Status | Recommendation |
|--------|-------------------------|-----------------|----------------|
| TI IWR6843AOP | ✅ Confirmed possible | Documented | **Primary choice** |
| TI IWR6843ISK | ✅ Confirmed possible | Documented | Primary for research |
| Infineon BGT60ATR24C | ⚠️ Likely | Research needed | Secondary |
| Acconeer XR112 | ❓ Unknown | Research needed | Low priority |

**For production deployment:** TI IWR6843AOP (antenna-on-package, compact form factor) with custom extended Doppler chirp configuration.

---

## 4. Detection Architecture

### 4.1 Continuous Wave (CW) Mode vs FMCW

**FMCW (Frequency-Modulated Continuous Wave):**
- Provides both range and velocity
- Requires chirp generation
- Processing complexity: Moderate
- Latency: Chirp duration + FFT processing

**CW (Continuous Wave) Mode:**
- Pure Doppler detection (velocity only, no range)
- Simpler hardware configuration
- Processing complexity: Low (single FFT)
- Latency: Only FFT processing time (~50-100 µs)

**Recommendation for extreme velocity detection:**

```
┌─────────────────────────────────────────────────────────────────┐
│                 EXTREME VELOCITY DETECTION MODE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CW Mode (Always-Active Background Scanning)                    │
│  ├── Zero scanning latency                                      │
│  ├── Continuous Doppler monitoring                              │
│  ├── Detects any fast-moving object                            │
│  ├── No range information                                       │
│  └── Latency: ~50-100 µs                                        │
│                                                                  │
│  FMCW Extended Doppler (When CW Triggers)                       │
│  ├── Provides range + velocity                                  │
│  ├── Higher latency (chirp + FFT ~500 µs)                       │
│  └── Used to characterize threat after CW detection             │
│                                                                  │
│  Event Camera (Simultaneous with CW/FMCW)                       │
│  ├── Microsecond trajectory visualization                      │
│  ├── Confirms direction of approach                            │
│  └── Provides visual evidence                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    DETECTION PIPELINE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [ CW Radar ] ──► Doppler FFT ──► Velocity Threshold          │
│       │                              │                           │
│       │                              ▼                           │
│       │                    [ v > 50 m/s? ]                      │
│       │                         │                                │
│       │                    YES  │  NO ──► Continue scanning     │
│       │                         ▼                                │
│       │                    TRIGGER                               │
│       │                         │                                │
│       ▼                         ▼                                │
│  [ Event Camera ] ────► Capture Frame                           │
│       │                         │                                │
│       ▼                         ▼                                │
│  [ FMCW Radar ] ────► Range + Velocity Characterization          │
│       │                         │                                │
│       │                         ▼                                │
│       │                    DETECTION EVENT                       │
│       │                         │                                │
│       │                         ▼                                │
│       └───────────────────► [ Transmit to Belt via BLE ]         │
│                                    │                             │
│                                    ▼                             │
│                              QoS = CRITICAL                      │
│                                    │                             │
│                                    ▼                             │
│                              Belt receives                       │
│                                    │                             │
│                                    ▼                             │
│                              Alert to wearer                    │
│                                                                  │
│  Timeline:                                                       │
│  ├── CW detection: 50-100 µs                                    │
│  ├── Event camera capture: < 1 ms                               │
│  ├── FMCW characterization: 500-1000 µs (optional)              │
│  ├── BLE transmission: 1-3 ms                                   │
│  ├── Belt processing: 100-500 µs                                │
│  ├── Haptic trigger: < 1 ms                                     │
│  └── Total: 2-6 ms (well within 3.5-8.5 ms window)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Velocity Thresholds

```toml
[extreme_velocity.detection]
# Velocity thresholds for triggering detection
min_velocity_ms = 50              # Ignore objects below 50 m/s
slow_projectile_threshold = 200   # 50-200 m/s = slow projectile
fast_projectile_threshold = 500   # 200-500 m/s = standard projectile
ultrafast_threshold = 800         # >500 m/s = high-velocity rifle

# Detection modes
mode = "cw_plus_event"            # "cw_only" | "cw_plus_event" | "cw_plus_fmcw" | "full"

# Processing configuration
doppler_fft_size = 256            # FFT size for Doppler analysis
doppler_update_rate_hz = 10000    # 10 kHz update rate
velocity_resolution_ms = 5        # Resolution (degraded for extended range)
```

---

## 5. Event Camera Integration

### 5.1 Role in Extreme Velocity Detection

**Event cameras provide:**
- Microsecond-latency motion detection
- Trajectory visualization (streak across pixel array)
- Direction of approach confirmation
- Visual evidence capture

**Why event cameras are essential:**
- Radar provides velocity but limited angular resolution
- Event camera provides precise direction and trajectory
- Fusion yields: velocity + direction + trajectory

### 5.2 Event Camera Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| Resolution | QVGA (320×240) minimum | Higher not required |
| Latency | < 1 ms to event output | Critical for detection |
| Event rate | 10+ M events/second | For fast-moving objects |
| Sensitivity | High | Detect small fast objects |

**Suitable event camera modules:**
- Sony IMX636-based modules
- Prophesee EVK3/EVK4 (evaluation)
- iniVation DAVIS346

### 5.3 Fusion with Doppler Radar

```
┌─────────────────────────────────────────────────────────────────┐
│                    SENSOR FUSION                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Doppler Radar:                                                  │
│  ├── Radial velocity (from Doppler shift)                       │
│  ├── Detection time (from FFT peak timestamp)                   │
│  └── Rough bearing (antenna array angle of arrival)             │
│                                                                  │
│  Event Camera:                                                   │
│  ├── Trajectory streak (pixel positions over time)              │
│  ├── Angular velocity (pixels per millisecond)                  │
│  ├── Direction of approach (streak direction)                   │
│  └── Time of arrival (event timestamp)                          │
│                                                                  │
│  Fusion Output:                                                  │
│  ├── Combined velocity vector (magnitude + direction)           │
│  ├── Predicted impact point (extrapolated trajectory)          │
│  ├── Time to impact (from current position)                    │
│  └── Object size estimate (from event density)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 Event Camera on 360° Pendant

The 360° pendant can integrate event cameras in addition to conventional cameras:

**Variant E — Event-Enhanced 360° Pendant:**
- 6 conventional cameras for visual 360° capture
- 2 event cameras at key angles (e.g., forward ±45°)
- Event cameras always-on for fast transient detection
- Conventional cameras activated for visual documentation

**Configuration:**

```toml
[nodes.pendant_360.event_cameras]
enabled = true
count = 2
positions = ["forward_left_45deg", "forward_right_45deg"]
always_on = true
trigger_threshold_events_per_ms = 1000  # Trigger when high event rate detected
```

---

## 6. Reaction Time Budget

### 6.1 Complete Timing Analysis

**Projectile approach (rifle, 850 m/s, 3 m distance):**

| Stage | Duration | Cumulative |
|-------|----------|------------|
| Object enters detection zone | T=0 | 0 |
| CW Doppler FFT processing | 50-100 µs | 100 µs |
| Velocity threshold check | < 10 µs | 110 µs |
| Event camera capture | < 1 ms | 1.1 ms |
| FMCW characterization (optional) | 500-1000 µs | 2.1 ms |
| Detection event generation | 10-50 µs | 2.15 ms |
| BLE queue (QoS = Critical, immediate) | 0-2 ms | 4.15 ms |
| BLE transmission | 1-3 ms | 7.15 ms |
| Belt reception and processing | 100-500 µs | 7.65 ms |
| Alert decision | 10-50 µs | 7.7 ms |
| Haptic trigger command | < 1 ms | 8.7 ms |
| **Total detection-to-alert** | **~5-9 ms** | |
| Object impact (if on trajectory) | 3.5 ms | |

**Critical insight:** The detection-to-alert time (~5-9 ms) EXCEEDS the time to impact (3.5 ms) for rifle projectiles at 3 m. This is the fundamental physics constraint.

**However:**
- At greater distances (5-10 m), time to impact increases to 6-12 ms
- Detection is useful for situational awareness and evidence capture
- Detection can trigger recording for forensic use
- Detection provides critical information even if interception is impossible

### 6.2 Latency Optimization Strategies

**Strategy 1: QoS = Critical Preemption**

```rust
// Detection event bypasses normal BLE scheduling
pub fn transmit_extreme_velocity_detection(event: ExtremeVelocityEvent) {
    let packet = BANPacket::new(QoS::Critical, event);
    // Critical packets preempt all other traffic
    ble_radio.transmit_immediate(packet);
}
```

**Impact:** Reduces BLE queue latency from 0-30 ms to 0-2 ms.

**Strategy 2: Reserved BLE Slot**

```
BLE Connection Event (30 ms interval)
├── Slot 0-2 ms:   Normal traffic
├── Slot 2-25 ms:  Normal traffic
├── Slot 25-28 ms: RESERVED (extreme velocity / critical alerts)
└── Slot 28-30 ms: Retransmits
```

**Impact:** Guarantees 3 ms slot every 30 ms for critical alerts.

**Strategy 3: Parallel Processing**

```
Timeline:
├── T=0 ms:     Doppler detection
├── T=0.1 ms:   Start event camera capture (parallel)
├── T=0.1 ms:   Start BLE transmission preparation (parallel)
├── T=1.1 ms:   Event camera data ready
├── T=1.1 ms:   BLE packet transmitted (QoS Critical)
├── T=4 ms:     Belt receives and processes
├── T=5 ms:     Haptic alert fires
```

**Impact:** Overlapping detection and transmission saves 1-2 ms.

---

## 7. Hardware Configuration

### 7.1 Radar Module Integration

**For pendant node (primary sensing position):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PENDANT NODE PCB                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │  mmWave Radar   │         │  Event Camera   │               │
│  │  (IWR6843AOP)   │◄────────│  (Optional)    │               │
│  │                 │  GPIO   │                 │               │
│  │  SPI Interface  │  Trigger│  MIPI/USB       │               │
│  └────────┬────────┘         └─────────────────┘               │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  Main MCU       │                                            │
│  │  (nRF5340)      │                                            │
│  │                 │                                            │
│  │  Radar driver   │                                            │
│  │  Doppler FFT    │                                            │
│  │  Event handler  │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  BLE Radio      │──► To belt node                           │
│  │  (Integrated)   │    QoS = Critical for extreme velocity    │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Pin Configuration

| Signal | Direction | MCU Pin | Description |
|--------|-----------|---------|-------------|
| `RADAR_CS` | Output | GPIO | Radar SPI chip select |
| `RADAR_IRQ` | Input | GPIO | Radar data ready interrupt |
| `RADAR_RST` | Output | GPIO | Radar hardware reset |
| `EVENT_CAM_TRIGGER` | Output | GPIO | Event camera enable/trigger |
| `EVENT_CAM_DATA` | Input | MIPI/SPI | Event camera data interface |

### 7.3 Antenna Considerations

**Antenna pattern for extreme velocity detection:**

| Requirement | Specification |
|-------------|---------------|
| Coverage | Forward hemisphere (primary threat direction) |
| Beam width | 60-90° azimuth, 30-45° elevation |
| Gain | 5-10 dBi typical |
| Mounting | Outward-facing from pendant (away from body) |

**Body shadowing mitigation:**
- Pendant is centered on torso (good forward visibility)
- Arms may partially obscure during movement
- Antenna diversity on belt node helps with link reliability

---

## 8. Firmware Implementation

### 8.1 Driver Architecture

```rust
// firmware/src/drivers/mmwave.rs

/// mmWave radar driver with extended Doppler support
pub struct MmWaveDriver {
    spi: SpiDevice,
    cs: OutputPin,
    irq: InputPin,
    reset: OutputPin,
    config: RadarConfig,
}

/// Radar configuration for different modes
pub struct RadarConfig {
    pub mode: RadarMode,
    pub chirp_config: Option<ChirpConfig>,
    pub update_rate_hz: u32,
    pub velocity_range_ms: f32,
}

pub enum RadarMode {
    Standard,           // Normal operation (±18 m/s)
    ExtendedDoppler,    // Projectile detection (±300 m/s)
    ContinuousWave,     // Pure Doppler (no range)
}

/// Extended Doppler chirp configuration
pub struct ChirpConfig {
    pub start_freq_hz: u64,
    pub chirp_slope_hz_per_us: f32,
    pub chirp_duration_us: u32,
    pub num_chirps_per_frame: u32,
    pub frame_period_us: u32,
}

impl MmWaveDriver {
    /// Configure radar for extreme velocity detection mode
    pub fn configure_extended_doppler(&mut self) -> Result<(), RadarError> {
        let config = ChirpConfig {
            start_freq_hz: 60_000_000_000,       // 60 GHz
            chirp_slope_hz_per_us: 10_000_000.0, // 10 MHz/µs (vs 40+ standard)
            chirp_duration_us: 10,                // 10 µs chirp
            num_chirps_per_frame: 1,              // Single chirp per frame
            frame_period_us: 100,                 // 100 µs frame = 10 kHz rate
        };
        
        self.write_chirp_config(&config)?;
        self.config.mode = RadarMode::ExtendedDoppler;
        self.config.velocity_range_ms = 300.0;
        
        Ok(())
    }
    
    /// Configure radar for continuous wave (pure Doppler) mode
    pub fn configure_continuous_wave(&mut self) -> Result<(), RadarError> {
        // CW mode: transmit continuous tone, measure Doppler shift only
        self.write_register(CW_MODE_ENABLE, 1)?;
        self.config.mode = RadarMode::ContinuousWave;
        self.config.velocity_range_ms = 500.0; // Extended range in CW
        
        Ok(())
    }
    
    /// Process incoming radar data and detect fast objects
    pub fn poll_detection(&mut self) -> Option<DetectionEvent> {
        if self.irq.is_high() {
            let raw_data = self.read_adc_samples()?;
            let doppler_spectrum = self.compute_doppler_fft(&raw_data);
            
            // Find peak velocity in Doppler spectrum
            let peak = self.find_velocity_peak(&doppler_spectrum);
            
            if peak.velocity_ms > self.config.min_velocity_threshold {
                return Some(DetectionEvent {
                    timestamp_us: self.get_timestamp(),
                    velocity_ms: peak.velocity_ms,
                    direction_rad: peak.angle_rad,
                    confidence: peak.snr_db,
                    event_type: EventType::ExtremeVelocity,
                });
            }
        }
        
        None
    }
    
    /// Compute Doppler FFT from raw ADC samples
    fn compute_doppler_fft(&self, samples: &[i16]) -> DopplerSpectrum {
        // Window function (Hamming or similar)
        let windowed: Vec<f32> = samples.iter()
            .enumerate()
            .map(|(i, &s)| s as f32 * hamming_window(i, samples.len()))
            .collect();
        
        // FFT computation (hardware accelerator or software)
        let spectrum = fft::compute(&windowed);
        
        // Convert to velocity bins
        DopplerSpectrum::from_fft(spectrum, self.config.velocity_range_ms)
    }
}
```

### 8.2 Event Camera Integration

```rust
// firmware/src/drivers/event_camera.rs

/// Event camera driver for fast transient detection
pub struct EventCameraDriver {
    interface: EventCameraInterface,
    config: EventCameraConfig,
}

pub struct EventCameraConfig {
    pub resolution: (u16, u16),
    pub sensitivity: f32,
    pub trigger_event_rate: u32,  // Events per ms to trigger detection
}

impl EventCameraDriver {
    /// Poll for high-event-rate conditions indicating fast object
    pub fn poll_fast_motion(&mut self) -> Option<FastMotionEvent> {
        let event_count = self.count_recent_events_ms(1); // Count events in last 1 ms
        
        if event_count > self.config.trigger_event_rate {
            // Fast motion detected
            let trajectory = self.compute_trajectory_from_events();
            
            return Some(FastMotionEvent {
                timestamp_us: self.get_timestamp(),
                event_count,
                direction_rad: trajectory.angle_rad,
                angular_velocity_rad_per_ms: trajectory.angular_velocity,
            });
        }
        
        None
    }
    
    /// Compute trajectory from event streak pattern
    fn compute_trajectory_from_events(&self) -> Trajectory {
        // Events from fast object form a streak across pixel array
        // Fit a line to recent events to determine direction
        let recent_events = self.get_recent_events(1000); // Last 1 ms
        let trajectory = fit_line_to_events(&recent_events);
        trajectory
    }
}
```

### 8.3 Fusion and Detection Event Generation

```rust
// firmware/src/logic/extreme_velocity_detection.rs

/// Extreme velocity detection fusion logic
pub struct ExtremeVelocityDetector {
    radar: MmWaveDriver,
    event_camera: Option<EventCameraDriver>,
    config: ExtremeVelocityConfig,
}

impl ExtremeVelocityDetector {
    /// Main detection loop
    pub fn poll(&mut self) -> Option<ExtremeVelocityEvent> {
        // Check CW Doppler radar
        let radar_detection = self.radar.poll_detection();
        
        // Check event camera (if present)
        let event_detection = self.event_camera.as_mut()
            .and_then(|cam| cam.poll_fast_motion());
        
        // Fuse detections
        match (radar_detection, event_detection) {
            (Some(radar), Some(event)) => {
                // Both sensors agree - high confidence
                Some(self.fuse_detections(radar, event))
            }
            (Some(radar), None) => {
                // Radar only - medium confidence
                Some(ExtremeVelocityEvent::from_radar_only(radar))
            }
            (None, Some(event)) => {
                // Event camera only - low confidence (no velocity)
                // Still generate event but mark as unconfirmed
                Some(ExtremeVelocityEvent::from_event_only(event, Confidence::Low))
            }
            (None, None) => None,
        }
    }
    
    /// Fuse radar and event camera detections
    fn fuse_detections(
        &self,
        radar: DetectionEvent,
        event: FastMotionEvent,
    ) -> ExtremeVelocityEvent {
        // Combine velocity from radar with direction from event camera
        let velocity_vector = VelocityVector {
            magnitude_ms: radar.velocity_ms,
            direction_rad: event.direction_rad,
        };
        
        ExtremeVelocityEvent {
            timestamp_us: radar.timestamp_us,
            velocity: velocity_vector,
            confidence: Confidence::High,
            sensors: SensorSet::RadarAndEvent,
        }
    }
    
    /// Generate BAN message for transmission to belt
    pub fn generate_ban_message(&self, event: ExtremeVelocityEvent) -> BANMessage {
        BANMessage::ExtremeVelocityDetection {
            timestamp_ms: event.timestamp_us / 1000,
            velocity_ms: event.velocity.magnitude_ms,
            direction_rad: event.velocity.direction_rad,
            confidence: event.confidence as u8,
            sensor_mask: event.sensors.bits(),
        }
    }
}
```

---

## 9. Configuration Reference

### 9.1 Complete Configuration

```toml
[extreme_velocity]
# Master enable
enabled = false

# Mode: "disabled" | "research" | "production"
mode = "production"

# Detection configuration
[extreme_velocity.detection]
# Velocity thresholds
min_velocity_ms = 50              # Ignore below 50 m/s
slow_projectile_threshold = 200   # Handgun range
fast_projectile_threshold = 500   # Rifle range
ultrafast_threshold = 800         # High-velocity rifle

# Detection modes
radar_mode = "continuous_wave"    # "continuous_wave" | "fmcw" | "hybrid"
event_camera_enabled = true
fusion_mode = "radar_and_event"   # "radar_only" | "event_only" | "radar_and_event"

# Processing parameters
doppler_fft_size = 256
doppler_update_rate_hz = 10000
velocity_resolution_ms = 5

# Detection window
max_detection_distance_m = 10
detection_field_of_view_deg = 90

# Alert configuration
[extreme_velocity.alert]
# Alert generation
generate_haptic_alert = true
generate_audio_alert = false
generate_app_alert = true

# Alert routing
alert_nodes = ["pendant", "bracelet_left", "bracelet_right"]
# Directional alerting - buzz node nearest to threat direction
directional_haptic = true

# Alert priority
qos_class = "critical"           # Preempts all other traffic

# Recording
[extreme_velocity.recording]
# Automatic recording on detection
record_event_camera = true
record_radar_burst = false
record_duration_s = 5.0

# Storage
storage_target = "sd_card"
include_integrity_chain = true

# Radar configuration
[extreme_velocity.radar]
# Extended Doppler chirp configuration
chirp_bandwidth_mhz = 500         # Reduced for velocity range
chirp_duration_us = 10
frame_period_us = 100             # 10 kHz update rate
velocity_range_ms = 300           # Max detectable velocity

# Event camera configuration
[extreme_velocity.event_camera]
resolution = "qvga"               # 320x240
sensitivity = 0.8
trigger_event_rate_per_ms = 1000
max_latency_us = 1000

# Timing and performance
[extreme_velocity.timing]
# Latency targets
max_total_latency_ms = 10
ble_qos = "critical"
ble_reserved_slot = true
ble_reserved_slot_start_ms = 25
ble_reserved_slot_end_ms = 28

# Performance monitoring
track_latency_stats = true
latency_warning_threshold_ms = 8
```

### 9.2 Mode Descriptions

**Mode: `disabled`**
- Extreme velocity detection completely off
- Radar operates in standard mode
- Event cameras (if present) operate normally
- No velocity thresholds applied

**Mode: `research`**
- Detection active but alerts muted
- Data logged for analysis
- Latency statistics recorded
- Useful for validating detection accuracy

**Mode: `production`**
- Full detection and alerting active
- All QoS mechanisms enabled
- Automatic recording on detection
- Alerts transmitted with Critical priority

---

## 10. Testing and Validation

### 10.1 Laboratory Testing

**Doppler velocity calibration:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Low velocity (50-100 m/s) | Compressed air projectile launcher | Detect within 100 µs |
| Medium velocity (200-300 m/s) | Airsoft gun (modified) | Detect within 200 µs |
| High velocity (800+ m/s) | Ballistic test range | Detect within 500 µs |

**Event camera calibration:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Fast motion streak | LED array on rotating wheel | Detect streak pattern |
| Direction accuracy | Known-angle projectile | Direction within ±10° |
| Latency | LED strobe + event capture | < 1 ms from event to detection |

### 10.2 Field Testing

**Scenario testing:**

1. **Standard detection scenario:**
   - Projectile launched at 250 m/s from 5 m
   - Expected detection within 2 ms
   - Alert to wearer within 8 ms

2. **Multi-sensor scenario:**
   - Radar + event camera both active
   - Cross-validate velocity and direction
   - Confirm fusion improves accuracy

3. **Body orientation scenario:**
   - Wearer in various positions
   - Confirm antenna diversity maintains link
   - Verify alerts reach belt node

### 10.3 Validation Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Detection latency | < 2 ms | Time from projectile entry to detection event |
| Alert latency | < 10 ms | Time from detection to haptic alert |
| Velocity accuracy | ±10% | Compare detected velocity to known value |
| Direction accuracy | ±15° | Compare detected direction to actual |
| False positive rate | < 1 per hour | Monitor during quiet periods |
| Miss rate | < 5% | Detect known projectiles |

---

## 11. Integration with 360° Pendant

### 11.1 Event-Enhanced 360° Pendant

The 360° pendant can integrate event cameras alongside conventional cameras for dual capability:

**Standard cameras (6-8):** Visual 360° capture for SLAM and recording

**Event cameras (2):** Fast transient detection for extreme velocity

**Configuration:**

```toml
[nodes.pendant_360]
# Standard cameras
camera_count = 8
camera_resolution = "720p"

# Event cameras (positioned at key angles)
event_camera_count = 2
event_camera_positions = ["forward_left", "forward_right"]

# Detection coordination
[extreme_velocity.pendant_360]
# Use event cameras for extreme velocity
use_event_cameras = true
# Trigger conventional cameras on detection
trigger_conventional_cameras = true
# Capture duration after detection
conventional_camera_capture_s = 2.0
```

### 11.2 Wiring Diagram

```
360° Pendant PCB (Event-Enhanced)
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Conv Cam 0  │  │ Conv Cam 1  │  │ Conv Cam 2  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│                          ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Vision Processor                        │ │
│  │                                                            │ │
│  │  [ Stitching Engine ]  [ Event Detection ]  [ Encoding ]  │ │
│  └────────────────────────────────────────────────────────────┘ │
│                          │                                      │
│         ┌────────────────┼────────────────┐                     │
│         │                │                │                     │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐             │
│  │ Event Cam L │  │ Radar Array │  │ Event Cam R │             │
│  │ (Forward)   │  │ (60 GHz)    │  │ (Forward)   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
│  Dedicated FSYNC line to all cameras                           │
│  ───────────────────────────────────────────────────────────────│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. Safety and Ethical Considerations

### 12.1 Not a Safety System

**Explicit disclaimers:**
- Extreme velocity detection is a sensing capability
- Detection does NOT prevent impact
- Time to impact often exceeds detection-to-alert time
- System provides situational awareness and evidence capture only
- Not a substitute for personal protective equipment
- Not a substitute for evasive action

### 12.2 Evidence Capture

**Legal value:**
- Detection events are timestamped and signed
- Event camera footage provides trajectory evidence
- Recordings include integrity chain for legal use
- All evidence stored locally (user-controlled)

### 12.3 Privacy

**Event camera data:**
- Event cameras produce motion patterns, not detailed imagery
- Streak patterns from projectiles don't capture faces
- Recording triggered only on detection events (not continuous)

---

## 13. Future Research Directions

### 13.1 Improved Velocity Range

**Goal:** Extend detection to > 1000 m/s (hypersonic fragments)

**Approach:**
- Higher carrier frequency (77 GHz automotive radar)
- Faster ADC sampling
- Shorter chirp duration

### 13.2 Multi-Antenna Direction Finding

**Goal:** Determine direction of approach from Doppler alone

**Approach:**
- Antenna array with beamforming
- Angle-of-arrival estimation
- Reduce reliance on event cameras

### 13.3 Machine Learning Classification

**Goal:** Distinguish projectile types from Doppler signature

**Approach:**
- Train classifier on Doppler signatures of different projectiles
- Identify caliber, shape, material from micro-Doppler
- Provide more detailed threat information

---

## 14. Summary

**Key conclusions:**

1. **Extended Doppler radar is viable** for projectile detection with custom chirp configuration
2. **TI IWR6843 is the primary candidate** with documented extended Doppler capability
3. **Event cameras provide essential complementary sensing** for trajectory and direction
4. **Fusion of radar + event camera** yields velocity + direction + trajectory
5. **Detection-to-alert latency of ~5-10 ms** is achievable with proper QoS configuration
6. **Critical QoS with reserved BLE slots** ensures reliable low-latency alert transmission
7. **Antenna diversity** improves link reliability for body-worn systems
8. **The architecture is production-capable** but detection does not prevent impact

---

**End of Document**
