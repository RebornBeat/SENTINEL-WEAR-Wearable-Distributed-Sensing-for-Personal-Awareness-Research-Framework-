# Doppler Radar Configuration for Extreme Velocity Detection

**Project:** SENTINEL-WEAR
**Domain:** High-speed object detection and threat characterization
**Status:** Production-Capable Architecture
**Implementation:** `crates/sentinel-extreme-velocity/`, `firmware/src/drivers/mmwave.rs`

---

## 1. Purpose

This document specifies the Doppler radar configuration architecture for detecting high-velocity objects (projectiles, debris, fragments) within the SENTINEL-WEAR body-frame awareness field. Unlike standard automotive or industrial radar profiles designed for human-scale velocities, extreme velocity detection requires extended Doppler range, high update rates, and microsecond-latency signal processing.

**Key thesis:** Standard mmWave radar profiles are optimized for human and vehicle motion (velocities up to ~18 m/s). Projectile detection requires velocities up to 850+ m/s, requiring custom chirp configurations that trade range resolution for velocity range.

**Critical architectural insight:** The detection pipeline must be separated into tiers with different latency requirements. The reflex trigger (Tier 1) can operate in hundreds of microseconds using CW radar. Full characterization (Tier 3) operates in milliseconds. These should not be conflated into a single latency budget.

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
| 850 m/s (rifle) | **340 kHz** |
| 1000 m/s (high-velocity) | 400 kHz |

**Critical insight:** Projectile Doppler signatures are MASSIVE compared to human motion. A walking person produces 400 Hz Doppler shift. A rifle bullet produces 340 kHz — nearly 1000× larger. This is spectrally trivial to detect with short FFT windows.

### 2.3 FFT Window Optimization for Extreme Velocities

**Doppler frequency resolution:**

$$\Delta f = \frac{1}{T}$$

Where T is the FFT integration window.

**Short window analysis:**

| FFT Window | Frequency Resolution | Velocity Resolution (60 GHz) | Usefulness |
|------------|---------------------|-----------------------------|------------|
| 32 µs | 31.25 kHz | ~77 m/s | **Excellent for threshold detection** |
| 64 µs | 15.625 kHz | ~39 m/s | Good for classification |
| 128 µs | 7.8 kHz | ~19 m/s | Detailed analysis |
| 256 µs | 3.9 kHz | ~10 m/s | Standard automotive |

**Key insight:** For extreme velocity detection, you do NOT need precise velocity measurement. You need to threshold: "something above 300 m/s exists."

A 32 µs FFT window provides 77 m/s resolution — coarse for precision measurement, but PERFECT for fast detection of extreme velocities. The detection can happen in tens of microseconds, not milliseconds.

### 2.4 Standard vs Extended Doppler Range

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

## 3. Tiered Detection Architecture

### 3.1 The Critical Architectural Insight

The original latency budget conflated multiple functions:
- Fast detection (reflex trigger)
- Direction validation
- Trajectory characterization
- Evidence recording
- Network transmission

These operate on vastly different timescales and should be architecturally separated.

### 3.2 Tier 1 — Reflex Trigger (~100-500 µs)

**Purpose:** Immediate detection of extreme velocity object, local alert trigger.

**Latency target:** 100-500 microseconds

**Sensors:** CW radar (primary), event camera (optional confirmation)

**Processing:**
- CW Doppler threshold detection
- No fusion required
- Local MCU decision
- Direct hardware trigger to haptic actuator

**Output:**
- Binary: "Extreme velocity object detected"
- Local haptic alert (if piezo actuator)
- Trigger signal to Tier 2 and Tier 3

**Data flow:**
```
CW Radar → Doppler FFT (32-64 µs) → Threshold Check (< 1 µs) → Hardware Trigger

Total: 50-150 µs electronic latency
```

### 3.3 Tier 2 — Direction Validation (~0.5-2 ms)

**Purpose:** Confirm detection, estimate direction of approach, prepare for characterization.

**Latency target:** 0.5-2 milliseconds

**Sensors:** Event camera (primary), CW radar confirmation

**Processing:**
- Event camera streak analysis
- Angle estimation from streak pattern
- Confidence scoring
- Fusion with radar velocity

**Output:**
- Direction vector (angle ± 15°)
- Velocity magnitude (from CW radar)
- Confidence level
- Alert routing decision (which node to buzz)

**Data flow:**
```
Tier 1 Trigger → Event Camera Readout (10-100 µs) → Streak Analysis (100-500 µs) →
Direction Estimate → BLE QoS Critical Transmission (300-800 µs) → Belt Reception

Total: 0.5-2 ms
```

### 3.4 Tier 3 — Characterization (~5-50 ms)

**Purpose:** Full threat characterization, evidence capture, logging, network transmission.

**Latency target:** 5-50 milliseconds (not time-critical)

**Sensors:** FMCW burst (range + velocity), event camera recording, conventional camera (if present)

**Processing:**
- FMCW range-Doppler analysis
- Trajectory prediction
- Object size estimation
- Evidence recording to SD card
- Network transmission to belt/app
- Logging with integrity chain

**Output:**
- Full threat characterization
- Recorded evidence
- Log entry with cryptographic hash
- Network notification

**Data flow:**
```
Tier 1 Trigger → FMCW Burst (500-1000 µs) → Point Cloud → Classification →
Recording → BLE Transmission → Belt Storage → Integrity Chain

Total: 5-50 ms
```

### 3.5 Why This Separation Matters

| Function | Old Approach | New Approach | Latency Improvement |
|----------|--------------|--------------|---------------------|
| Detection | FMCW frame wait + processing | CW threshold | 10-50× faster |
| Alert | Full pipeline before alert | Tier 1 triggers local alert | Immediate vs delayed |
| Characterization | Same path as detection | Separate Tier 3 | Non-blocking |
| Evidence | Integrated with detection | Post-trigger recording | No detection delay |

**The Tier 1 reflex trigger can fire before the projectile has traveled 0.5 meters.**

---

## 4. CW Radar vs FMCW — The Primary Trigger Decision

### 4.1 Why CW Radar Is the Primary Trigger

**FMCW latency budget:**
- Chirp generation: 5-20 µs
- Chirp duration: 8-15 µs
- ADC sampling: 8-15 µs
- Range FFT: 20-100 µs
- Doppler FFT: 20-100 µs
- Detection processing: 10-50 µs
- **Total: 70-300 µs minimum**

**CW radar latency budget:**
- Continuous transmission (no chirp wait)
- ADC sampling: 32-64 µs (short window sufficient)
- Doppler FFT: 20-50 µs
- Threshold check: < 1 µs
- **Total: 50-120 µs**

**CW radar advantages:**
- No chirp timing overhead
- No range FFT required (velocity only)
- Shorter integration windows acceptable (large Doppler shifts)
- Always-on, zero startup latency
- Simpler firmware

**FMCW role:**
- Tier 3 characterization only
- Provides range + refined velocity
- Not on critical detection path

### 4.2 CW Radar Configuration

```c
// CW radar configuration for 60 GHz module
cw_config = {
    .carrier_frequency = 60e9,           // 60 GHz
    .tx_power_dbm = 10,                   // Maximum allowed
    
    // Receiver configuration
    .if_gain_db = 40,                     // High gain for weak returns
    .adc_sample_rate = 2e6,               // 2 MSPS sufficient
    .fft_size = 64,                       // Short FFT for fast processing
    .fft_window_us = 32,                  // 32 µs window = 31.25 kHz resolution
    
    // Detection thresholds
    .velocity_threshold_ms = 50,          // Ignore below 50 m/s
    .snr_threshold_db = 10,               // Minimum signal-to-noise
    
    // Update rate
    .detection_interval_us = 100,         // 10 kHz detection rate
};
```

### 4.3 CW + Event Camera Fusion (Tier 1 + Tier 2)

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIER 1 + TIER 2 FUSION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CW Radar (Primary Trigger):                                    │
│  ├── Continuous Doppler monitoring                              │
│  ├── FFT window: 32-64 µs                                      │
│  ├── Detection latency: 50-150 µs                               │
│  ├── Output: velocity magnitude, SNR                            │
│  └── Trigger: "something fast exists"                           │
│                                                                  │
│  Event Camera (Direction Confirmation):                          │
│  ├── Asynchronous event emission                                │
│  ├── Event latency: 10-100 µs (microsecond-scale)              │
│  ├── Streak analysis: 100-500 µs                                │
│  ├── Output: direction angle, angular velocity                  │
│  └── Trigger: confirms radar detection                          │
│                                                                  │
│  Fusion Output:                                                  │
│  ├── Velocity: from CW radar (magnitude only)                   │
│  ├── Direction: from event camera streak                         │
│  ├── Combined: velocity vector (magnitude + direction)          │
│  ├── Confidence: radar SNR × streak quality                     │
│  └── Latency: 100-500 µs total                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. mmWave Module Capabilities

### 5.1 TI IWR6843 Family

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

**CW mode on IWR6843:**

The IWR6843 can operate in continuous wave mode for pure Doppler detection:

| Parameter | CW Mode Value |
|-----------|---------------|
| Latency | 50-150 µs |
| Velocity range | ±500 m/s (with appropriate processing) |
| Range information | None |
| Update rate | Up to 100 kHz |
| Power | Lower than FMCW |

**Implementation:**

```c
// TI mmWave SDK configuration for CW mode
// Simplified pseudo-code

cw_config = {
    .mode = CONTINUOUS_WAVE,
    .carrier_frequency = 60e9,
    
    // No chirp configuration needed in CW mode
    .tx_power = 10,  // dBm
    
    // Receiver
    .if_bandwidth_hz = 500e3,        // 500 kHz IF bandwidth
    .adc_sample_rate = 2e6,          // 2 MSPS
    
    // Processing
    .fft_size = 64,                  // Short FFT for speed
    .detection_threshold_snr = 10,    // dB
    
    // Continuous operation
    .frame_trigger = AUTO_CONTINUOUS,
};
```

### 5.2 Infineon BGT60ATR24C

**Overview:** 60 GHz MMIC radar, SPI interface, compact form factor.

**Characteristics:**
- Fully programmable chirp
- CW mode supported
- High integration density
- Compact form factor ideal for pendant

**Extended Doppler capability:**
- Configurable via SPI
- Research needed for optimal parameters
- Promising for jewelry form factor

### 5.3 Acconeer XR112

**Overview:** 60 GHz pulse radar, ultra-low power, SPI interface.

**Characteristics:**
- Pulse radar (different from FMCW)
- Excellent low-power presence detection
- Single-shot velocity measurement possible
- May have advantages for extreme velocity

**Suitability:** Requires research. Pulse radar may detect single fast events efficiently.

### 5.4 Module Selection Matrix

| Module | CW Mode | Extended FMCW | Form Factor | Power | Recommendation |
|--------|---------|---------------|-------------|-------|-----------------|
| TI IWR6843AOP | ✅ Supported | ✅ Supported | Integrated antenna | Moderate | **Primary choice** |
| TI IWR6843ISK | ✅ Supported | ✅ Supported | Module + external | Higher | Research platform |
| Infineon BGT60ATR24C | ✅ Supported | ⚠️ Research needed | MMIC (compact) | Low | Secondary |
| Acconeer XR112 | ⚠️ Different principle | N/A | Ultra-compact | Ultra-low | Research |

---

## 6. Event Camera Integration

### 6.1 Why Event Cameras Are Essential

**Event cameras provide:**
- Microsecond-latency motion detection (10-100 µs, not 1 ms)
- Trajectory visualization (streak across pixel array)
- Direction of approach confirmation
- No frame wait — asynchronous event emission

**Timing reality:**

| Component | Latency | Notes |
|-----------|---------|-------|
| Event generation (sensor) | 10-50 µs | Photon to event |
| Event transmission | 10-50 µs | To MCU via MIPI/SPI |
| Streak detection | 50-200 µs | Pattern recognition |
| Angle estimation | 50-200 µs | Line fitting |
| **Total** | **100-500 µs** | Direction validation |

This is much faster than the original `< 1 ms` estimate because event cameras have no frame latency — events appear continuously.

### 6.2 Event Camera Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT CAMERA PROCESSING                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [ Event Sensor ]                                               │
│  ├── Asynchronous event emission                                │
│  ├── Events queued in circular buffer                           │
│  └── DMA transfer to MCU memory                                 │
│                                                                  │
│  [ Streak Detection ]                                           │
│  ├── Monitor recent event density                               │
│  ├── Threshold: events/ms > X → fast motion detected           │
│  └── Trigger: collect streak events                             │
│                                                                  │
│  [ Direction Estimation ]                                        │
│  ├── Collect events in 100-500 µs window                        │
│  ├── Fit line to event positions                                │
│  ├── Line angle = direction of approach                         │
│  └── Angular velocity = streak length / time                    │
│                                                                  │
│  [ Output ]                                                      │
│  ├── Direction angle: θ ± 15°                                   │
│  ├── Angular velocity: dθ/dt                                    │
│  └── Confidence: based on streak quality                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Event Camera Parameters for Extreme Velocity

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Resolution | QVGA (320×240) | Higher not needed for streak detection |
| Latency | 10-100 µs | Asynchronous, no frame wait |
| Event rate capacity | 10+ M events/sec | Fast objects generate bursts |
| Sensitivity | High | Detect small fast objects |
| Region of interest | Forward hemisphere | Primary threat direction |

### 6.4 Event Camera on 360° Pendant

**Variant E — Event-Enhanced 360° Pendant:**

The 360° pendant can integrate event cameras alongside conventional cameras:

| Camera Type | Count | Purpose |
|-------------|-------|---------|
| Conventional | 6-8 | Visual 360° capture, SLAM |
| Event | 2 | Fast transient detection, direction |

**Positioning:**
- Event cameras at forward angles (±45° from center)
- Cover primary threat approach direction
- Conventional cameras cover full 360° for evidence

**Configuration:**

```toml
[nodes.pendant_360.event_cameras]
enabled = true
count = 2
positions = ["forward_left_45deg", "forward_right_45deg"]

[nodes.pendant_360.event_cameras.detection]
# Trigger parameters
streak_event_threshold_per_ms = 500    # Events/ms to trigger
min_streak_length_px = 20              # Minimum streak length
max_detection_latency_us = 500         # Target latency

# Processing
dma_enabled = true                      # Direct memory access
interrupt_driven = true                 # Event-triggered ISR
```

---

## 7. Realistic Latency Budgets

### 7.1 Conservative Production System

| Stage | Latency | Notes |
|-------|---------|-------|
| CW Doppler detection | 50-150 µs | FFT + threshold |
| Event camera confirmation | 100-500 µs | Streak analysis |
| Local MCU decision | 5-50 µs | Classification |
| BLE QoS Critical transmission | 300-800 µs | Optimized stack |
| Belt reception | 50-150 µs | IRQ-driven |
| **Total electronic latency** | **0.5-2.0 ms** | Before actuator |

### 7.2 Haptic Actuator Physics

**The actuator is the bottleneck for human-perceptible alerting:**

| Actuator Type | Onset Time | Suitability |
|---------------|------------|-------------|
| ERM motor | 10-50 ms | ❌ Too slow |
| LRA (linear resonant) | 2-10 ms | ⚠️ Marginal |
| Piezo haptic | 0.1-1.0 ms | ✅ Excellent |
| Electrostatic skin stimulator | < 0.1 ms | ✅ Best (if available) |

**Recommendation:** For extreme velocity alerting, piezo haptic actuators are mandatory. Standard vibration motors are too slow to be useful within the detection window.

### 7.3 Complete End-to-End Timing

**With piezo haptics:**

| Stage | Latency |
|-------|---------|
| Tier 1 detection (CW radar) | 50-150 µs |
| Tier 2 direction (event camera) | 100-500 µs |
| BLE transmission | 300-800 µs |
| Belt processing | 50-150 µs |
| Piezo haptic trigger | 50-300 µs |
| Piezo mechanical onset | 100-500 µs |
| **Total** | **0.65-2.4 ms** |

**With standard LRA haptics:**

| Stage | Latency |
|-------|---------|
| All electronic | 0.5-1.5 ms |
| LRA onset | 2-10 ms |
| **Total** | **2.5-11.5 ms** |

### 7.4 Comparison to Projectile Flight Time

| Projectile | Velocity | Time to 3 m | Detection + Piezo Alert | Detection + LRA Alert |
|------------|----------|--------------|------------------------|----------------------|
| Rifle (850 m/s) | 850 m/s | 3.5 ms | 0.65-2.4 ms ✅ | 2.5-11.5 ms ⚠️ |
| Handgun (350 m/s) | 350 m/s | 8.5 ms | 0.65-2.4 ms ✅ | 2.5-11.5 ms ✅ |

**With piezo haptics, alert fires before rifle bullet reaches wearer at 3 m distance.**

**With LRA haptics, alert fires near or after rifle bullet impact.**

---

## 8. BLE Latency Optimization

### 8.1 Why Standard BLE Is Too Slow

Standard BLE implementations assume:
- Generic GATT profile
- Pairing negotiation
- Connection interval negotiation
- Application-layer buffering
- OS scheduler involvement

For SENTINEL-WEAR, the belt node and sensing nodes are controlled devices with:
- Pre-paired relationships
- Fixed connection intervals
- Direct IRQ-driven processing
- No OS scheduler (embedded, not Linux)

### 8.2 Optimized BLE Stack

**Optimizations:**

| Optimization | Latency Impact |
|--------------|----------------|
| Pre-established connections | Eliminates connection setup |
| Fixed connection interval (7.5 ms) | Predictable latency |
| QoS = Critical preemption | Bypass queue |
| Direct ISR to radio | No buffering |
| Nordic-to-Nordic fast path | Hardware-optimized |
| Reserved timeslot | Guaranteed transmission |

**Optimized BLE latency:**

| Scenario | Latency |
|----------|---------|
| Standard BLE (worst case) | 0-30 ms |
| Optimized BLE (QoS Critical) | 0.3-0.8 ms |
| Reserved slot + QoS Critical | 0.2-0.5 ms |

### 8.3 BLE Configuration

```toml
[ban.ble_optimization]
# Low-latency BLE configuration
connection_interval_ms = 7.5           # Minimum allowed
slave_latency = 0                      # No skipped events
supervision_timeout_ms = 1000

# QoS configuration
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
# Extreme velocity detection is Critical
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 1
reserved_slot_enabled = true
reserved_slot_start_ms = 5
reserved_slot_end_ms = 7

# Firmware behavior
direct_isr_to_radio = true             # Bypass application buffer
zero_copy_transmission = true          # DMA from detection to radio
```

---

## 9. Firmware Implementation

### 9.1 CW Radar Driver

```rust
// firmware/src/drivers/mmwave_cw.rs

/// CW radar driver for extreme velocity detection
pub struct CwRadarDriver {
    spi: SpiDevice,
    config: CwRadarConfig,
    fft_buffer: [Complex<f32>; 64],
    velocity_threshold_ms: f32,
}

pub struct CwRadarConfig {
    pub carrier_freq_hz: u64,
    pub if_gain_db: u8,
    pub adc_sample_rate_hz: u32,
    pub fft_size: usize,
    pub fft_window_us: u32,
    pub velocity_threshold_ms: f32,
    pub snr_threshold_db: f32,
}

impl CwRadarDriver {
    /// Configure radar for CW mode
    pub fn init(&mut self) -> Result<(), RadarError> {
        // Enable CW mode on radar module
        self.write_register(REG_MODE, MODE_CW)?;
        self.write_register(REG_TX_POWER, 10)?; // 10 dBm
        self.write_register(REG_IF_GAIN, self.config.if_gain_db)?;
        
        Ok(())
    }
    
    /// Poll for extreme velocity detection
    /// Returns immediately if no detection, or detection event within ~100 µs
    pub fn poll_detection(&mut self) -> Option<ExtremeVelocityEvent> {
        // Read ADC samples (DMA transfer, ~32-64 µs)
        let samples = self.read_adc_samples_dma();
        
        // Apply window function
        self.apply_window(&samples);
        
        // Compute FFT (hardware accelerator or optimized software)
        // 64-point FFT on Cortex-M7 with FPU: ~20-50 µs
        let spectrum = self.compute_fft(&self.fft_buffer);
        
        // Find peak in spectrum
        let peak = self.find_peak(&spectrum);
        
        // Convert frequency to velocity
        let velocity_ms = self.freq_to_velocity(peak.freq_hz);
        
        // Check threshold
        if velocity_ms > self.velocity_threshold_ms && peak.snr_db > self.config.snr_threshold_db {
            return Some(ExtremeVelocityEvent {
                timestamp_us: self.get_timestamp_us(),
                velocity_ms,
                snr_db: peak.snr_db,
                confidence: Confidence::High,
            });
        }
        
        None
    }
    
    /// Convert Doppler frequency to velocity
    fn freq_to_velocity(&self, freq_hz: f32) -> f32 {
        // v = (Δf × c) / (2 × f_c)
        (freq_hz * 3e8) / (2.0 * self.config.carrier_freq_hz as f32)
    }
    
    /// 64-point FFT optimized for ARM Cortex-M7
    fn compute_fft(&mut self, buffer: &mut [Complex<f32>; 64]) -> Spectrum {
        // Use CMSIS-DSP or arm_cfft_f32 for optimal speed
        // 64-point FFT on Cortex-M7 @ 480 MHz: ~15-25 µs
        arm_cfft_f32(buffer);
        
        // Convert to magnitude spectrum
        let mut spectrum = Spectrum::default();
        for (i, bin) in buffer.iter().enumerate() {
            spectrum.bins[i] = (bin.re * bin.re + bin.im * bin.im).sqrt();
        }
        spectrum
    }
}

/// Detection event from CW radar
pub struct ExtremeVelocityEvent {
    pub timestamp_us: u64,
    pub velocity_ms: f32,
    pub snr_db: f32,
    pub confidence: Confidence,
}
```

### 9.2 Event Camera Driver

```rust
// firmware/src/drivers/event_camera.rs

/// Event camera driver optimized for fast transient detection
pub struct EventCameraDriver {
    interface: EventInterface,
    event_buffer: CircularBuffer<Event, 4096>,
    streak_threshold: u32,
    detection_callback: Option<fn(StreakEvent)>,
}

impl EventCameraDriver {
    /// Poll for fast motion streak
    /// Called from ISR when events arrive
    pub fn process_events(&mut self) -> Option<StreakEvent> {
        // Read events from sensor (already DMA'd to buffer)
        let recent_count = self.count_events_in_window_us(500); // Last 500 µs
        
        // Fast motion threshold
        if recent_count > self.streak_threshold {
            // Collect streak events
            let streak_events = self.get_recent_events(500);
            
            // Fit line to streak
            let direction = self.fit_line_to_events(&streak_events);
            
            return Some(StreakEvent {
                timestamp_us: self.get_timestamp_us(),
                direction_rad: direction.angle,
                angular_velocity: direction.angular_velocity,
                event_count: recent_count,
                confidence: self.calculate_confidence(&streak_events),
            });
        }
        
        None
    }
    
    /// Fit line to event positions for direction estimation
    fn fit_line_to_events(&self, events: &[Event]) -> Direction {
        if events.len() < 10 {
            return Direction::unknown();
        }
        
        // Simple linear regression on x,y coordinates
        let n = events.len() as f32;
        let sum_x: f32 = events.iter().map(|e| e.x as f32).sum();
        let sum_y: f32 = events.iter().map(|e| e.y as f32).sum();
        let sum_xy: f32 = events.iter().map(|e| (e.x as f32) * (e.y as f32)).sum();
        let sum_x2: f32 = events.iter().map(|e| (e.x as f32).powi(2)).sum();
        
        let denom = n * sum_x2 - sum_x * sum_x;
        if denom.abs() < 1e-6 {
            return Direction::unknown();
        }
        
        let slope = (n * sum_xy - sum_x * sum_y) / denom;
        let angle = slope.atan2(1.0); // Direction in radians
        
        // Angular velocity from streak length over time
        let first_t = events.first().unwrap().timestamp_us;
        let last_t = events.last().unwrap().timestamp_us;
        let duration_us = last_t.saturating_sub(first_t);
        
        // Calculate streak length in pixels
        let first = (events.first().unwrap().x, events.first().unwrap().y);
        let last = (events.last().unwrap().x, events.last().unwrap().y);
        let length_px = ((last.0 - first.0).pow(2) + (last.1 - first.1).pow(2)) as f32;
        let angular_velocity = if duration_us > 0 {
            length_px / (duration_us as f32 / 1000.0) // pixels per ms
        } else {
            0.0
        };
        
        Direction {
            angle,
            angular_velocity,
        }
    }
}
```

### 9.3 Fusion and Detection Pipeline

```rust
// firmware/src/logic/extreme_velocity_pipeline.rs

/// Complete extreme velocity detection pipeline
pub struct ExtremeVelocityPipeline {
    cw_radar: CwRadarDriver,
    event_camera: EventCameraDriver,
    ble_transmitter: BleTransmitter,
    haptic_driver: HapticDriver,
    
    // Configuration
    config: PipelineConfig,
    
    // State
    tier1_triggered: bool,
    tier2_result: Option<Tier2Result>,
}

impl ExtremeVelocityPipeline {
    /// Main detection loop — called from main loop or ISR
    pub fn poll(&mut self) {
        // === TIER 1: Reflex Trigger (50-150 µs) ===
        if let Some(detection) = self.cw_radar.poll_detection() {
            self.tier1_triggered = true;
            
            // Immediately trigger local haptic (if piezo)
            if self.config.piezo_haptic_enabled {
                self.haptic_driver.trigger_immediate();
            }
            
            // Start Tier 2 analysis
            self.event_camera.start_capture();
        }
        
        // === TIER 2: Direction Validation (100-500 µs after Tier 1) ===
        if self.tier1_triggered {
            if let Some(streak) = self.event_camera.poll_detection() {
                // Fuse radar velocity with camera direction
                self.tier2_result = Some(Tier2Result {
                    velocity: self.cw_radar.last_velocity(),
                    direction: streak.direction_rad,
                    confidence: streak.confidence,
                });
                
                // Transmit to belt with QoS Critical
                self.ble_transmit_detection(&self.tier2_result);
            }
        }
    }
    
    /// Transmit detection to belt node
    fn ble_transmit_detection(&mut self, result: &Option<Tier2Result>) {
        if let Some(data) = result {
            let packet = BANPacket::new(
                QoS::Critical,
                ExtremeVelocityMessage {
                    timestamp_us: self.get_timestamp_us(),
                    velocity_ms: data.velocity,
                    direction_rad: data.direction,
                    confidence: data.confidence as u8,
                },
            );
            
            // QoS Critical bypasses normal queue
            self.ble_transmitter.transmit_immediate(packet);
        }
    }
}
```

---

## 10. Hardware Configuration

### 10.1 Radar Module Integration

**For pendant node (primary sensing position):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PENDANT NODE PCB                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │  CW Radar       │         │  Event Camera   │               │
│  │  (IWR6843AOP)   │◄────────│  (IMX636)       │               │
│  │                 │  GPIO   │                 │               │
│  │  SPI Interface  │  Trigger│  MIPI/USB       │               │
│  └────────┬────────┘         └─────────────────┘               │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │  Main MCU       │         │  Piezo Haptic   │               │
│  │  (nRF5340)      │────────►│  (DRV2667)      │               │
│  │                 │  GPIO   │                 │               │
│  │  CW driver      │         │  Sub-ms onset   │               │
│  │  Event handler  │         │                 │               │
│  └────────┬────────┘         └─────────────────┘               │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  BLE Radio      │──► To belt node (QoS Critical)           │
│  │  (Integrated)   │                                            │
│  └─────────────────┘                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Pin Configuration

| Signal | Direction | MCU Pin | Description |
|--------|-----------|---------|-------------|
| `RADAR_CS` | Output | GPIO | Radar SPI chip select |
| `RADAR_IRQ` | Input | GPIO | Radar data ready interrupt |
| `RADAR_RST` | Output | GPIO | Radar hardware reset |
| `EVENT_DATA` | Input | MIPI/SPI | Event camera data |
| `EVENT_TRIGGER` | Output | GPIO | Event camera enable |
| `PIEZO_PWM` | Output | PWM | Piezo haptic driver |
| `PIEZO_EN` | Output | GPIO | Piezo enable |

### 10.3 Piezo Haptic Driver Requirements

| Parameter | Value | Notes |
|-----------|-------|-------|
| Driver IC | DRV2667 or equivalent | Piezo driver with boost |
| Response time | < 500 µs | Onset time |
| Drive voltage | 50-200 Vpp | Typical piezo |
| Waveform | Single impulse | For alert |
| Connection | I2C + PWM | Configuration + drive |

---

## 11. Configuration Reference

### 11.1 Complete Configuration

```toml
[extreme_velocity]
# Master enable
enabled = false

# Mode: "disabled" | "research" | "production"
mode = "production"

# Tiered architecture configuration
[extreme_velocity.tier1]
# Reflex trigger parameters
radar_mode = "continuous_wave"      # "continuous_wave" | "fmcw"
velocity_threshold_ms = 50          # Ignore below 50 m/s
detection_latency_target_us = 150   # Target: 150 µs
local_haptic_trigger = true         # Fire local alert immediately
haptic_type = "piezo"               # "piezo" | "lra"

# CW radar parameters
[extreme_velocity.tier1.cw_radar]
fft_size = 64
fft_window_us = 32
adc_sample_rate_hz = 2000000
if_gain_db = 40
velocity_resolution_ms = 77          # For 32 µs window at 60 GHz

# Event camera parameters
[extreme_velocity.tier1.event_camera]
enabled = true
streak_threshold_per_ms = 500
min_streak_length_px = 20
capture_window_us = 500

# Tier 2: Direction validation
[extreme_velocity.tier2]
enabled = true
direction_accuracy_deg = 15
fusion_mode = "radar_and_event"
latency_target_ms = 2

# Tier 3: Characterization
[extreme_velocity.tier3]
enabled = true
fmcw_burst = true
record_evidence = true
latency_target_ms = 50

# BLE configuration
[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true
reserved_slot_start_ms = 5
reserved_slot_end_ms = 7
direct_isr_to_radio = true

# Haptic configuration
[extreme_velocity.haptic]
type = "piezo"
onset_time_ms = 0.5
pattern = "single_impulse"

# Alert routing
[extreme_velocity.alert]
directional = true                    # Alert node nearest to threat
nodes = ["pendant", "bracelet_left", "bracelet_right"]

# Recording configuration
[extreme_velocity.recording]
trigger_conventional_cameras = true
capture_duration_s = 2.0
include_integrity_chain = true
```

---

## 12. Testing and Validation

### 12.1 Laboratory Testing

**Doppler velocity calibration:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Low velocity (50-100 m/s) | Compressed air launcher | Detect within 200 µs |
| Medium velocity (200-300 m/s) | Airsoft gun (modified) | Detect within 150 µs |
| High velocity (800+ m/s) | Ballistic test range | Detect within 100 µs |

**Latency measurement:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Tier 1 latency | LED strobe + sensor | < 200 µs |
| Tier 2 latency | Known projectile | < 2 ms |
| BLE transmission | Logic analyzer | < 1 ms |
| Piezo onset | High-speed camera | < 500 µs |

### 12.2 End-to-End Testing

**Scenario: Rifle projectile (850 m/s) at 5 m**

| Metric | Target | Notes |
|--------|--------|-------|
| Detection time | < 200 µs | Tier 1 CW radar |
| Direction estimate | < 1 ms | Tier 2 event camera |
| Alert transmission | < 1 ms | BLE QoS Critical |
| Haptic onset | < 500 µs | Piezo actuator |
| **Total alert latency** | **< 2 ms** | |
| Projectile flight time | 5.9 ms | Time to traverse 5 m |

**Result:** Alert fires ~4 ms before impact, wearer has ~4 ms of warning.

### 12.3 Validation Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Tier 1 latency | < 200 µs | Oscilloscope on detection output |
| Tier 2 latency | < 2 ms | Timestamp comparison |
| Velocity accuracy | ±10% | Compare to chronograph |
| Direction accuracy | ±15° | Known-angle projectile |
| False positive rate | < 1 per hour | Monitor during quiet periods |
| Miss rate | < 5% | Known projectile test |

---

## 13. Integration with 360° Pendant

### 13.1 Event-Enhanced 360° Pendant Configuration

```toml
[nodes.pendant_360]
# Standard cameras for 360° visual capture
camera_count = 8
camera_resolution = "720p"

# Event cameras for extreme velocity detection
event_camera_count = 2
event_camera_positions = ["forward_left_45deg", "forward_right_45deg"]

# CW radar for primary trigger
cw_radar = true
cw_radar_position = "center"

# Piezo haptic for local alert
piezo_haptic = true

# Detection configuration
[extreme_velocity.pendant_360]
use_event_cameras = true
trigger_conventional_cameras = true
conventional_camera_capture_s = 2.0
```

---

## 14. Safety and Ethical Considerations

### 14.1 Fundamental Physics Constraint

**Detection does NOT prevent impact.**

Even with optimized latency:
- Detection + piezo alert: ~0.65-2.4 ms
- Rifle bullet at 850 m/s travels: 0.55-2.0 meters during alert

**At 3 m engagement distance:**
- Flight time: 3.5 ms
- Alert latency: 0.65-2.4 ms
- Warning time: 1.1-2.85 ms

**This provides situational awareness and reflex preparation, not bullet dodging.**

### 14.2 What the System Actually Provides

| Capability | Reality |
|------------|---------|
| Extreme velocity detection | ✅ Yes — microsecond-scale |
| Direction awareness | ✅ Yes — within 1-2 ms |
| Alert to wearer | ✅ Yes — piezo haptic |
| Evidence capture | ✅ Yes — camera recording |
| Impact prevention | ❌ No — physics impossible |
| Evasive action | ⚠️ Limited — depends on human reaction time |

### 14.3 Disclaimers

- Not a certified safety system
- Not personal protective equipment
- Not a substitute for body armor
- Detection capability does not imply protective capability
- User assumes all responsibility for deployment

---

## 15. Summary

**Key conclusions:**

1. **Tiered architecture separates concerns by latency:**
   - Tier 1 (Reflex Trigger): 100-500 µs using CW radar
   - Tier 2 (Direction Validation): 0.5-2 ms using event camera
   - Tier 3 (Characterization): 5-50 ms for evidence and logging

2. **CW radar is the primary trigger** — no chirp latency, pure Doppler, microsecond response

3. **Projectile Doppler signatures are massive** — 340 kHz for rifle, trivial to detect with short FFT windows

4. **Event cameras have microsecond latency** — 10-100 µs, not 1 ms as originally estimated

5. **Piezo haptics are mandatory** — LRA motors (2-10 ms onset) are too slow; piezo (0.1-1 ms) is required

6. **BLE can be optimized** — QoS Critical with reserved slots achieves 0.3-0.8 ms

7. **Total alert latency of 0.65-2.4 ms is achievable** — alerts fire within flight time for most scenarios

8. **Detection does not prevent impact** — provides awareness and evidence, not protection

---

**End of Document**
