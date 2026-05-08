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

**Corrected understanding:** While close-range scenarios (3-15 m) present tight latency constraints, realistic rifle engagements occur at 50-300 meters where flight times are 60-350 ms. At these distances, detection range is the primary constraint, not alert latency. Standard BLE and LRA haptics provide sufficient warning for the majority of realistic scenarios.

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

### 2.2 Realistic Engagement Distance Analysis

**Critical correction:** The 3-meter rifle scenario is point-blank range — extremely rare in actual shooting incidents and physically unavoidable regardless of detection technology. Realistic engagement distances provide substantially more warning time.

#### Handgun Engagements

| Distance | Velocity | Flight Time | Detection + Alert | Warning Time |
|----------|----------|--------------|-------------------|--------------|
| 3 m | 350 m/s | 8.5 ms | 0.65-2.4 ms | **6.1-7.85 ms** |
| 5 m | 350 m/s | 14.3 ms | 0.65-2.4 ms | **11.9-13.65 ms** |
| 10 m | 350 m/s | 28.6 ms | 0.65-2.4 ms | **26.2-27.95 ms** |
| 15 m | 350 m/s | 42.9 ms | 0.65-2.4 ms | **40.5-42.25 ms** |

#### Rifle Engagements (Realistic Distances)

| Distance | Velocity | Flight Time | Detection + Alert | Warning Time |
|----------|----------|--------------|-------------------|--------------|
| 50 m | 850 m/s | 58.8 ms | 0.65-2.4 ms | **56.4-58.15 ms** |
| 100 m | 850 m/s | 117.6 ms | 0.65-2.4 ms | **115.2-116.95 ms** |
| 200 m | 850 m/s | 235.3 ms | 0.65-2.4 ms | **232.9-234.65 ms** |
| 300 m | 850 m/s | 352.9 ms | 0.65-2.4 ms | **350.5-352.25 ms** |

**Key insight:** At 100 meters (common active shooter scenario), the wearer has **115-117 milliseconds of warning** — well within human reaction time (~200 ms for simple reactions). At 300 meters, warning time exceeds **350 milliseconds**.

### 2.3 Implications for System Requirements

| Requirement | Close-Range (< 15 m) | Medium-Range (15-50 m) | Long-Range (50-300 m) |
|-------------|---------------------|------------------------|----------------------|
| Latency optimization | Critical | Important | Comfortable margins |
| Piezo haptics | Valuable | Beneficial | Optional |
| BLE optimization | Valuable | Beneficial | Optional |
| Detection range | Secondary | Important | **Primary** |
| Evidence capture | Always important | Always important | Always important |

### 2.4 Doppler Radar Fundamentals

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

### 2.5 FFT Window Optimization for Extreme Velocities

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

### 2.6 Standard vs Extended Doppler Range

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

## 3. Detection Range Optimization

### 3.1 The Primary Constraint

**Detection range > Alert latency.**

For rifle engagements at 100+ meters, flight time (117-353 ms) vastly exceeds any realistic alert latency. The critical question becomes: at what distance can the system detect the projectile?

### 3.2 Detection Range vs Flight Time

| Detection Range | Rifle Flight Time to Wearer | Warning Time Available |
|-----------------|-----------------------------|------------------------|
| 3 m | 3.5 ms | 1-2 ms (tight) |
| 5 m | 5.9 ms | 3-5 ms |
| 10 m | 11.8 ms | 9-11 ms |
| 20 m | 23.5 ms | 21-23 ms |
| 50 m | 58.8 ms | 56-58 ms |
| 100 m | 117.6 ms | 115-117 ms |
| 150 m | 176.5 ms | 174-176 ms |

### 3.3 Detection Range Configuration

**Range-optimized configuration:**

```toml
[extreme_velocity.detection_range]
mode = "max_range"

[extreme_velocity.detection_range.radar]
tx_power_dbm = 12                 # Higher transmit power
antenna_gain_dbi = 15             # Directional antenna for pendant
target_range_m = 150              # Detect at 150+ meters
range_resolution_m = 2.0          # Accept coarse resolution
velocity_range_ms = 1000          # Support up to 1000 m/s

[extreme_velocity.detection_range.latency]
# Latency is relaxed; flight time provides margin
fft_window_us = 128               # Longer window for sensitivity
latency_budget_ms = 5             # Acceptable at 150 m (176 ms flight time)
```

**Latency-optimized configuration (close-range scenarios):**

```toml
[extreme_velocity.detection_latency]
mode = "min_latency"

[extreme_velocity.detection_latency.radar]
fft_window_us = 32                # Shortest window
velocity_resolution_ms = 77       # Coarse but fast
latency_budget_us = 150           # Target: 150 µs

[extreme_velocity.detection_latency.haptic]
type = "piezo"                    # Fastest onset
onset_ms = 0.5
```

**Balanced configuration:**

```toml
[extreme_velocity.balanced]
mode = "balanced"

[extreme_velocity.balanced.radar]
target_range_m = 100
range_resolution_m = 0.5
velocity_resolution_ms = 50
latency_budget_ms = 2

[extreme_velocity.balanced.haptic]
type = "lra"                      # Sufficient for 100+ m
onset_ms = 5
```

---

## 4. Tiered Detection Architecture

### 4.1 The Critical Architectural Insight

The detection pipeline must separate functions operating on different timescales:
- Fast detection (reflex trigger): microseconds
- Direction validation: hundreds of microseconds
- Trajectory characterization: milliseconds
- Evidence recording: tens of milliseconds
- Network transmission: tens of milliseconds

These should not be conflated into a single latency budget.

### 4.2 Tier 1 — Reflex Trigger (~50-150 µs)

**Purpose:** Immediate detection of extreme velocity object, local alert trigger.

**Latency target:** 50-150 microseconds

**Sensors:** CW radar (primary), event camera (optional confirmation)

**Processing:**
- CW Doppler threshold detection
- No fusion required
- Local MCU decision
- Direct hardware trigger to haptic actuator

**Output:**
- Binary: "Extreme velocity object detected"
- Local haptic alert (piezo preferred, LRA acceptable for 50+ m)
- Trigger signal to Tier 2 and Tier 3

**Data flow:**
```
CW Radar → Doppler FFT (32-64 µs) → Threshold Check (< 1 µs) → Hardware Trigger

Total: 50-150 µs electronic latency
```

**Context at realistic distances:**
- Rifle at 100 m: Flight time = 117.6 ms
- Tier 1 latency = 0.05-0.15 ms
- Tier 1 consumes **0.04-0.13%** of available warning time

### 4.3 Tier 2 — Direction Validation (~0.1-0.5 ms)

**Purpose:** Confirm detection, estimate direction of approach, prepare for characterization.

**Latency target:** 100-500 microseconds

**Sensors:** Event camera (primary), CW radar confirmation

**Event camera timing reality:**

| Component | Latency |
|-----------|---------|
| Event generation (sensor) | 10-50 µs |
| Event transmission | 10-50 µs |
| Streak detection | 50-200 µs |
| Angle estimation | 50-200 µs |
| **Total** | **100-500 µs** |

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
Direction Estimate → BLE Transmission → Belt Reception

Total: 0.5-2 ms
```

**Context at realistic distances:**
- Rifle at 100 m: Flight time = 117.6 ms
- Tier 1 + Tier 2 = 0.6-2.15 ms
- Tiers 1+2 consume **0.5-1.8%** of available warning time

### 4.4 Tier 3 — Characterization (~5-50 ms)

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

**Context at realistic distances:**
- Rifle at 100 m: Flight time = 117.6 ms
- Full pipeline (Tiers 1+2+3) = 5.6-52.15 ms
- Complete pipeline consumes **4.8-44%** of available warning time
- **Still completes before impact at 100+ meters**

### 4.5 Tiered Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 EXTREME VELOCITY DETECTION TIERS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TIER 1 — REFLEX TRIGGER                                                     │
│  Latency: 50-150 µs                                                          │
│  Sensors: CW radar (primary)                                                 │
│  Processing: Doppler FFT threshold                                           │
│  Output: Binary detection, local haptic trigger                              │
│  Purpose: Immediate awareness, startle response                              │
│  ────────────────────────────────────────────────────────────────────────── │
│  Notes: Trivial latency compared to any realistic flight time                │
│         Triggers Tier 2 and Tier 3                                           │
│         At 100 m: consumes 0.04-0.13% of warning time                         │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TIER 2 — DIRECTION VALIDATION                                               │
│  Latency: 100-500 µs (plus event camera: 10-100 µs)                           │
│  Sensors: Event camera (primary), CW radar confirmation                      │
│  Processing: Streak analysis, angle estimation                               │
│  Output: Velocity vector (magnitude + direction)                             │
│  Purpose: Directional alert routing                                          │
│  ────────────────────────────────────────────────────────────────────────── │
│  Notes: Still trivial compared to realistic flight time                      │
│         Provides actionable direction information                            │
│         At 100 m: Tiers 1+2 consume 0.5-1.8% of warning time                  │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TIER 3 — CHARACTERIZATION                                                   │
│  Latency: 5-50 ms                                                            │
│  Sensors: FMCW burst, event camera recording, conventional cameras           │
│  Processing: Range-Doppler analysis, trajectory, evidence capture            │
│  Output: Full threat characterization, recorded evidence                     │
│  Purpose: Evidence, logging, forensic analysis                               │
│  ────────────────────────────────────────────────────────────────────────── │
│  Notes: Non-blocking; does not delay Tier 1 or Tier 2                        │
│         At 100 m: completes well before impact                                │
│         Full pipeline (Tiers 1+2+3): 5.6-52 ms = 4.8-44% of warning time      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.6 Why This Separation Matters

| Function | Old Approach | New Approach | Latency Improvement |
|----------|--------------|--------------|---------------------|
| Detection | FMCW frame wait + processing | CW threshold | 10-50× faster |
| Alert | Full pipeline before alert | Tier 1 triggers local alert | Immediate vs delayed |
| Characterization | Same path as detection | Separate Tier 3 | Non-blocking |
| Evidence | Integrated with detection | Post-trigger recording | No detection delay |

---

## 5. CW Radar vs FMCW — The Primary Trigger Decision

### 5.1 Why CW Radar Is the Primary Trigger

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

### 5.2 CW Radar Configuration

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

### 5.3 Range-Optimized CW Configuration

For detection at 150+ meters:

```c
// Range-optimized CW radar configuration
cw_range_config = {
    .carrier_frequency = 60e9,
    .tx_power_dbm = 12,                   // Higher power
    
    // Receiver optimized for sensitivity
    .if_gain_db = 50,                     // Maximum gain
    .adc_sample_rate = 1e6,               // 1 MSPS for sensitivity
    .fft_size = 128,                      // Longer FFT for sensitivity
    .fft_window_us = 128,                 // 128 µs window
    
    // Detection thresholds (relaxed for range)
    .velocity_threshold_ms = 100,
    .snr_threshold_db = 6,                // Lower threshold
    
    // Tradeoffs accepted
    .velocity_resolution_ms = 19,          // Coarser resolution
    .latency_budget_us = 200,              // Accept longer latency
};
```

### 5.4 CW + Event Camera Fusion (Tier 1 + Tier 2)

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIER 1 + TIER 2 FUSION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CW Radar (Primary Trigger):                                    │
│  ├── Continuous Doppler monitoring                              │
│  ├── FFT window: 32-64 µs (fast) or 128 µs (sensitive)         │
│  ├── Detection latency: 50-150 µs (fast) or 200 µs (sensitive)  │
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

## 6. mmWave Module Capabilities

### 6.1 TI IWR6843 Family

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
| Latency | 50-150 µs (fast) or 200 µs (sensitive) |
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
    .tx_power = 10,  // dBm (or 12 for range-optimized)
    
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

### 6.2 Infineon BGT60ATR24C

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

### 6.3 Acconeer XR112

**Overview:** 60 GHz pulse radar, ultra-low power, SPI interface.

**Characteristics:**
- Pulse radar (different from FMCW)
- Excellent low-power presence detection
- Single-shot velocity measurement possible
- May have advantages for extreme velocity

**Suitability:** Requires research. Pulse radar may detect single fast events efficiently.

### 6.4 Module Selection Matrix

| Module | CW Mode | Extended FMCW | Form Factor | Power | Recommendation |
|--------|---------|---------------|-------------|-------|-----------------|
| TI IWR6843AOP | ✅ Supported | ✅ Supported | Integrated antenna | Moderate | **Primary choice** |
| TI IWR6843ISK | ✅ Supported | ✅ Supported | Module + external | Higher | Research platform |
| Infineon BGT60ATR24C | ✅ Supported | ⚠️ Research needed | MMIC (compact) | Low | Secondary |
| Acconeer XR112 | ⚠️ Different principle | N/A | Ultra-compact | Ultra-low | Research |

---

## 7. Event Camera Integration

### 7.1 Why Event Cameras Are Essential

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

### 7.2 Event Camera Processing Pipeline

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

### 7.3 Event Camera Parameters for Extreme Velocity

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Resolution | QVGA (320×240) | Higher not needed for streak detection |
| Latency | 10-100 µs | Asynchronous, no frame wait |
| Event rate capacity | 10+ M events/sec | Fast objects generate bursts |
| Sensitivity | High | Detect small fast objects |
| Region of interest | Forward hemisphere | Primary threat direction |

### 7.4 Event Camera on 360° Pendant

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

## 8. Haptic Actuator Selection

### 8.1 The Critical Question

**Original conclusion:** Piezo haptics mandatory for extreme velocity detection

**Corrected understanding:** Haptic selection depends on engagement distance

### 8.2 Haptic Response Times

| Actuator Type | Onset Time | Peak Output | Cost | Complexity |
|---------------|------------|-------------|------|------------|
| ERM motor | 10-50 ms | Strong | Low | Low |
| LRA (linear resonant) | 2-10 ms | Moderate | Medium | Medium |
| Piezo haptic | 0.1-1.0 ms | Light-moderate | High | High |
| Electrostatic skin stimulator | < 0.1 ms | Very light | Very high | Very high |

### 8.3 Scenario-Based Haptic Selection

| Scenario | Flight Time | Recommended Haptic | Warning Time |
|----------|-------------|-------------------|--------------|
| Rifle at 50 m | 58.8 ms | LRA sufficient | 48-56 ms ✅ |
| Rifle at 100 m | 117.6 ms | LRA sufficient | 107-115 ms ✅ |
| Rifle at 200 m | 235.3 ms | Any haptic works | 225-233 ms ✅ |
| Handgun at 5 m | 14.3 ms | Piezo valuable | 12-13 ms ✅ |
| Handgun at 3 m | 8.5 ms | Piezo preferred | 6-7 ms ⚠️ |

**Conclusion:**
- **LRA sufficient for rifle at 50+ meters** (majority of realistic scenarios)
- **Piezo valuable for close-range handgun** (< 10 m)
- **Piezo not mandatory** for typical rifle engagements

### 8.4 Recommended Configuration

```toml
[haptic]
# Primary haptic type
type = "lra"                      # "lra" | "piezo" | "both"

# Piezo as optional enhancement
[haptic.piezo]
enabled = false                   # Populated on PCB, disabled by default
onset_ms = 0.5
use_for = "close_range_only"      # "always" | "close_range_only" | "extreme_velocity_only"

# LRA configuration
[haptic.lra]
onset_ms = 5
pattern_library = ["single_pulse", "double_pulse", "triple_pulse"]

# Directional routing
[haptic.directional]
enabled = true
actuator_count = 4                # Front, back, left, right
routing = "nearest_to_threat"
```

---

## 9. BLE Latency Analysis

### 9.1 Standard vs Optimized BLE

| BLE Mode | Worst-Case Latency | Use Case |
|----------|-------------------|----------|
| Standard (30 ms interval) | 0-30 ms | Rifle at 50+ m ✅ |
| Optimized (QoS Critical) | 0.3-0.8 ms | Close-range scenarios |
| Reserved slot | 0.2-0.5 ms | Maximum margin |

### 9.2 BLE Selection by Scenario

| Scenario | Flight Time | Standard BLE Warning | Optimized BLE Warning |
|----------|-------------|---------------------|----------------------|
| Rifle at 50 m | 58.8 ms | 28-58 ms ✅ | 58-58 ms ✅ |
| Rifle at 100 m | 117.6 ms | 87-117 ms ✅ | 116-117 ms ✅ |
| Rifle at 200 m | 235.3 ms | 205-235 ms ✅ | 234-235 ms ✅ |
| Handgun at 10 m | 28.6 ms | 0-28 ms ⚠️ | 27-28 ms ✅ |
| Handgun at 5 m | 14.3 ms | 0-14 ms ❌ | 13-14 ms ⚠️ |

**Conclusion:**
- **Standard BLE sufficient for rifle at 50+ meters**
- **Optimized BLE valuable for handgun at < 15 m**
- **BLE optimization is beneficial but not mandatory for typical rifle scenarios**

### 9.3 BLE Configuration

```toml
[ban.ble_extreme_velocity]
# For most scenarios, standard BLE provides adequate latency
mode = "standard"                 # "standard" | "optimized" | "adaptive"

# Adaptive mode: use optimized BLE only for close-range detections
[ban.ble_extreme_velocity.adaptive]
enable_for_close_range = true
close_range_threshold_m = 20     # Use optimized BLE for detections within 20 m

[ban.ble_optimization]
connection_interval_ms = 7.5
slave_latency = 0
supervision_timeout_ms = 1000

[ban.qos.critical]
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 1
reserved_slot_enabled = true
reserved_slot_start_ms = 5
reserved_slot_end_ms = 7

direct_isr_to_radio = true
zero_copy_transmission = true
```

---

## 10. Realistic End-to-End Latency Budgets

### 10.1 Conservative Production System (All Components)

| Stage | Latency | Notes |
|-------|---------|-------|
| CW Doppler detection | 50-150 µs | FFT + threshold |
| Event camera confirmation | 100-500 µs | Streak analysis |
| Local MCU decision | 5-50 µs | Classification |
| BLE transmission (standard) | 0-30 ms | Variable |
| BLE transmission (optimized) | 300-800 µs | QoS Critical |
| Belt reception | 50-150 µs | IRQ-driven |
| LRA haptic onset | 2-10 ms | Standard haptic |
| Piezo haptic onset | 0.1-1.0 ms | Fast haptic |

### 10.2 Complete Timing by Configuration

**Configuration A — Standard (LRA + Standard BLE):**

| Stage | Latency |
|-------|---------|
| Detection + direction | 0.2-0.7 ms |
| BLE transmission | 0-30 ms |
| Haptic onset | 2-10 ms |
| **Total** | **2.2-40.7 ms** |

**Configuration B — Optimized (LRA + Optimized BLE):**

| Stage | Latency |
|-------|---------|
| Detection + direction | 0.2-0.7 ms |
| BLE transmission | 0.3-0.8 ms |
| Haptic onset | 2-10 ms |
| **Total** | **2.5-11.5 ms** |

**Configuration C — Maximum Speed (Piezo + Optimized BLE):**

| Stage | Latency |
|-------|---------|
| Detection + direction | 0.2-0.7 ms |
| BLE transmission | 0.3-0.8 ms |
| Piezo onset | 0.1-1.0 ms |
| **Total** | **0.6-2.5 ms** |

### 10.3 Scenario Analysis with Realistic Configurations

**Rifle at 100 meters:**

| Configuration | Total Alert Latency | Flight Time | Warning Time | Assessment |
|---------------|---------------------|-------------|--------------|------------|
| Standard (LRA + Std BLE) | 2.2-40.7 ms | 117.6 ms | 76.9-115.4 ms | ✅ Excellent |
| Optimized (LRA + Opt BLE) | 2.5-11.5 ms | 117.6 ms | 106.1-115.1 ms | ✅ Excellent |
| Maximum Speed | 0.6-2.5 ms | 117.6 ms | 115.1-117.0 ms | ✅ Excellent |

**Rifle at 50 meters:**

| Configuration | Total Alert Latency | Flight Time | Warning Time | Assessment |
|---------------|---------------------|-------------|--------------|------------|
| Standard | 2.2-40.7 ms | 58.8 ms | 18.1-56.6 ms | ✅ Good |
| Optimized | 2.5-11.5 ms | 58.8 ms | 47.3-56.3 ms | ✅ Excellent |
| Maximum Speed | 0.6-2.5 ms | 58.8 ms | 56.3-58.2 ms | ✅ Excellent |

**Handgun at 10 meters:**

| Configuration | Total Alert Latency | Flight Time | Warning Time | Assessment |
|---------------|---------------------|-------------|--------------|------------|
| Standard | 2.2-40.7 ms | 28.6 ms | 0-26.4 ms | ⚠️ Variable |
| Optimized | 2.5-11.5 ms | 28.6 ms | 17.1-26.1 ms | ✅ Good |
| Maximum Speed | 0.6-2.5 ms | 28.6 ms | 26.1-28.0 ms | ✅ Good |

**Handgun at 5 meters:**

| Configuration | Total Alert Latency | Flight Time | Warning Time | Assessment |
|---------------|---------------------|-------------|--------------|------------|
| Standard | 2.2-40.7 ms | 14.3 ms | 0-12.1 ms | ❌ Unreliable |
| Optimized | 2.5-11.5 ms | 14.3 ms | 2.8-11.8 ms | ⚠️ Tight |
| Maximum Speed | 0.6-2.5 ms | 14.3 ms | 11.8-13.7 ms | ⚠️ Useful |

---

## 11. Firmware Implementation

### 11.1 CW Radar Driver

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
    pub range_optimized: bool,      // For long-range detection
}

impl CwRadarDriver {
    /// Configure radar for CW mode
    pub fn init(&mut self) -> Result<(), RadarError> {
        // Enable CW mode on radar module
        self.write_register(REG_MODE, MODE_CW)?;
        self.write_register(REG_TX_POWER, 
            if self.config.range_optimized { 12 } else { 10 })?;
        self.write_register(REG_IF_GAIN, self.config.if_gain_db)?;
        
        Ok(())
    }
    
    /// Poll for extreme velocity detection
    /// Returns immediately if no detection, or detection event within ~100 µs (fast)
    /// or ~200 µs (range-optimized)
    pub fn poll_detection(&mut self) -> Option<ExtremeVelocityEvent> {
        // Read ADC samples (DMA transfer)
        let samples = self.read_adc_samples_dma();
        
        // Apply window function
        self.apply_window(&samples);
        
        // Compute FFT
        let spectrum = self.compute_fft(&mut self.fft_buffer);
        
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
        (freq_hz * 3e8) / (2.0 * self.config.carrier_freq_hz as f32)
    }
    
    /// FFT computation optimized for ARM Cortex-M7
    fn compute_fft(&mut self, buffer: &mut [Complex<f32>; 64]) -> Spectrum {
        // CMSIS-DSP 64-point FFT on Cortex-M7 @ 480 MHz: ~15-25 µs
        arm_cfft_f32(buffer);
        
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

### 11.2 Event Camera Driver

```rust
// firmware/src/drivers/event_camera.rs

/// Event camera driver optimized for fast transient detection
pub struct EventCameraDriver {
    interface: EventInterface,
    event_buffer: CircularBuffer<Event, 4096>,
    streak_threshold: u32,
}

impl EventCameraDriver {
    /// Poll for fast motion streak
    /// Called from ISR when events arrive
    pub fn process_events(&mut self) -> Option<StreakEvent> {
        // Count events in recent window
        let recent_count = self.count_events_in_window_us(500);
        
        // Fast motion threshold
        if recent_count > self.streak_threshold {
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
    
    /// Fit line to event positions
    fn fit_line_to_events(&self, events: &[Event]) -> Direction {
        if events.len() < 10 {
            return Direction::unknown();
        }
        
        // Linear regression on x,y coordinates
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
        let angle = slope.atan2(1.0);
        
        // Angular velocity from streak length
        let first = events.first().unwrap();
        let last = events.last().unwrap();
        let duration_us = last.timestamp_us.saturating_sub(first.timestamp_us);
        let length_px = ((last.x - first.x).pow(2) + (last.y - first.y).pow(2)) as f32;
        let angular_velocity = if duration_us > 0 {
            length_px / (duration_us as f32 / 1000.0)
        } else { 0.0 };
        
        Direction { angle, angular_velocity }
    }
}
```

### 11.3 Fusion Pipeline

```rust
// firmware/src/logic/extreme_velocity_pipeline.rs

/// Complete extreme velocity detection pipeline
pub struct ExtremeVelocityPipeline {
    cw_radar: CwRadarDriver,
    event_camera: EventCameraDriver,
    ble_transmitter: BleTransmitter,
    haptic_driver: HapticDriver,
    config: PipelineConfig,
    tier1_triggered: bool,
    tier2_result: Option<Tier2Result>,
}

impl ExtremeVelocityPipeline {
    /// Main detection loop
    pub fn poll(&mut self) {
        // TIER 1: Reflex Trigger
        if let Some(detection) = self.cw_radar.poll_detection() {
            self.tier1_triggered = true;
            
            // Immediate local haptic (if piezo)
            if self.config.piezo_haptic_enabled {
                self.haptic_driver.trigger_immediate();
            }
            
            self.event_camera.start_capture();
        }
        
        // TIER 2: Direction Validation
        if self.tier1_triggered {
            if let Some(streak) = self.event_camera.poll_detection() {
                self.tier2_result = Some(Tier2Result {
                    velocity: self.cw_radar.last_velocity(),
                    direction: streak.direction_rad,
                    confidence: streak.confidence,
                });
                
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
    
    /// Get current timestamp in microseconds
    fn get_timestamp_us(&self) -> u64 {
        // Use hardware timer or PTP-synchronized clock
        unsafe { (*TIM2::ptr()).cnt.read().bits() as u64 }
    }
}

/// Result from Tier 2 fusion
pub struct Tier2Result {
    pub velocity: f32,
    pub direction: f32,
    pub confidence: f32,
}

/// Pipeline configuration
pub struct PipelineConfig {
    pub piezo_haptic_enabled: bool,
    pub ble_qos_critical: bool,
    pub tier2_timeout_us: u32,
    pub tier3_enabled: bool,
}

/// Extreme velocity message for BAN transmission
#[derive(Serialize)]
pub struct ExtremeVelocityMessage {
    pub timestamp_us: u64,
    pub velocity_ms: f32,
    pub direction_rad: f32,
    pub confidence: u8,
}
```

---

## 12. Hardware Configuration

### 12.1 Radar Module Integration

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

### 12.2 Pin Configuration

| Signal | Direction | MCU Pin | Description |
|--------|-----------|---------|-------------|
| `RADAR_CS` | Output | GPIO | Radar SPI chip select |
| `RADAR_IRQ` | Input | GPIO | Radar data ready interrupt |
| `RADAR_RST` | Output | GPIO | Radar hardware reset |
| `EVENT_DATA` | Input | MIPI/SPI | Event camera data |
| `EVENT_TRIGGER` | Output | GPIO | Event camera enable |
| `PIEZO_PWM` | Output | PWM | Piezo haptic driver |
| `PIEZO_EN` | Output | GPIO | Piezo enable |

### 12.3 Piezo Haptic Driver Requirements

| Parameter | Value | Notes |
|-----------|-------|-------|
| Driver IC | DRV2667 or equivalent | Piezo driver with boost |
| Response time | < 500 µs | Onset time |
| Drive voltage | 50-200 Vpp | Typical piezo |
| Waveform | Single impulse | For alert |
| Connection | I2C + PWM | Configuration + drive |

### 12.4 Long-Range Detection Hardware (Variant F)

**For maximum detection range (150+ m):**

| Component | Specification | Notes |
|-----------|---------------|-------|
| CW radar | IWR6843ISK with external antenna | Higher gain antenna |
| Tx power | 12 dBm | Maximum allowed |
| Antenna gain | 15 dBi | Directional patch or horn |
| Beam width | 30-60° | Narrower for gain |
| Detection range target | 150 m | Against rifle projectile |

**Tradeoffs accepted:**
- Reduced field of view (directional)
- Larger form factor
- Higher power consumption
- Coarser velocity resolution (for sensitivity)

---

## 13. Configuration Reference

### 13.1 Complete Configuration

```toml
[extreme_velocity]
# Master enable
enabled = false

# Mode: "disabled" | "research" | "production"
mode = "production"

# Detection optimization target
optimization_target = "balanced"   # "range" | "latency" | "balanced"

# Tiered architecture configuration
[extreme_velocity.tier1]
# Reflex trigger parameters
radar_mode = "continuous_wave"      # "continuous_wave" | "fmcw"
velocity_threshold_ms = 50          # Ignore below 50 m/s
detection_latency_target_us = 150   # Target: 150 µs
local_haptic_trigger = true         # Fire local alert immediately
haptic_type = "lra"                 # "piezo" | "lra" | "both"

# CW radar parameters
[extreme_velocity.tier1.cw_radar]
fft_size = 64
fft_window_us = 32                  # 32 µs for fast, 128 µs for sensitive
adc_sample_rate_hz = 2000000
if_gain_db = 40                     # 40 dB standard, 50 dB for range-optimized
velocity_resolution_ms = 77         # For 32 µs window at 60 GHz

# Range-optimized configuration
[extreme_velocity.tier1.cw_radar.range_optimized]
enabled = false
target_range_m = 150
fft_window_us = 128
if_gain_db = 50
velocity_resolution_ms = 19

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
# For most scenarios, standard BLE is sufficient
mode = "adaptive"                   # "standard" | "optimized" | "adaptive"

[extreme_velocity.ble.adaptive]
# Use optimized BLE for close-range detections
enable_for_close_range = true
close_range_threshold_m = 20

[extreme_velocity.ble.optimized]
qos_class = "critical"
reserved_slot = true
reserved_slot_start_ms = 5
reserved_slot_end_ms = 7
direct_isr_to_radio = true

# Haptic configuration
[extreme_velocity.haptic]
type = "lra"                        # "lra" | "piezo" | "both"
onset_time_ms = 5                   # LRA: 2-10 ms, Piezo: 0.5-1 ms
pattern = "single_impulse"

# Piezo optional enhancement
[extreme_velocity.haptic.piezo]
enabled = false                     # Populated but disabled by default
use_for = "close_range_only"        # "always" | "close_range_only" | "extreme_velocity_only"

# Alert routing
[extreme_velocity.alert]
directional = true                  # Alert node nearest to threat
nodes = ["pendant", "bracelet_left", "bracelet_right", "anklet_left", "anklet_right"]

# Recording configuration
[extreme_velocity.recording]
trigger_conventional_cameras = true
capture_duration_s = 2.0
include_integrity_chain = true

# Detection range configuration
[extreme_velocity.detection_range]
# Trade range vs latency
mode = "balanced"                   # "range_optimized" | "latency_optimized" | "balanced"

[extreme_velocity.detection_range.balanced]
target_range_m = 100
range_resolution_m = 0.5
velocity_resolution_ms = 50
latency_budget_ms = 2

[extreme_velocity.detection_range.range_optimized]
target_range_m = 150
range_resolution_m = 2.0
velocity_resolution_ms = 150
latency_budget_ms = 5

[extreme_velocity.detection_range.latency_optimized]
target_range_m = 50
range_resolution_m = 0.3
velocity_resolution_ms = 20
latency_budget_ms = 1
```

### 13.2 Scenario-Based Configuration Examples

**Urban concealed carry scenario (close-range handgun focus):**

```toml
[extreme_velocity]
enabled = true
mode = "production"
optimization_target = "latency"

[extreme_velocity.tier1.cw_radar]
fft_window_us = 32              # Fastest detection

[extreme_velocity.haptic]
type = "piezo"                  # Piezo for close-range

[extreme_velocity.ble]
mode = "optimized"              # Minimize latency
```

**Open area security scenario (long-range rifle focus):**

```toml
[extreme_velocity]
enabled = true
mode = "production"
optimization_target = "range"

[extreme_velocity.tier1.cw_radar.range_optimized]
enabled = true
target_range_m = 150

[extreme_velocity.haptic]
type = "lra"                    # LRA sufficient for 100+ m

[extreme_velocity.ble]
mode = "standard"               # Standard BLE sufficient
```

**General consumer scenario (balanced):**

```toml
[extreme_velocity]
enabled = true
mode = "production"
optimization_target = "balanced"

[extreme_velocity.haptic]
type = "lra"                    # LRA sufficient for most scenarios

[extreme_velocity.ble]
mode = "adaptive"               # Standard for far, optimized for close
```

---

## 14. Testing and Validation

### 14.1 Laboratory Testing

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
| BLE transmission (standard) | Logic analyzer | < 30 ms |
| BLE transmission (optimized) | Logic analyzer | < 1 ms |
| Piezo onset | High-speed camera | < 500 µs |
| LRA onset | High-speed camera | < 10 ms |

### 14.2 End-to-End Testing

**Scenario: Rifle projectile (850 m/s) at realistic distances**

| Distance | Flight Time | Standard Config Warning | Optimized Config Warning |
|----------|-------------|------------------------|--------------------------|
| 50 m | 58.8 ms | 18.1-56.6 ms ✅ | 56.3-58.2 ms ✅ |
| 100 m | 117.6 ms | 76.9-115.4 ms ✅ | 115.1-117.0 ms ✅ |
| 200 m | 235.3 ms | 194.6-233.1 ms ✅ | 232.8-234.7 ms ✅ |

**Scenario: Handgun projectile (350 m/s) at realistic distances**

| Distance | Flight Time | Standard Config Warning | Optimized Config Warning |
|----------|-------------|------------------------|--------------------------|
| 5 m | 14.3 ms | 0-12.1 ms ⚠️ | 11.8-13.7 ms ⚠️ |
| 10 m | 28.6 ms | 0-26.4 ms ⚠️ | 26.1-28.0 ms ✅ |
| 15 m | 42.9 ms | 2.2-40.7 ms ✅ | 40.4-42.3 ms ✅ |

### 14.3 Validation Metrics

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Tier 1 latency | < 200 µs | Oscilloscope on detection output |
| Tier 2 latency | < 2 ms | Timestamp comparison |
| Detection range (rifle) | ≥ 100 m | Known projectile test at 100 m |
| Detection range (handgun) | ≥ 15 m | Known projectile test at 15 m |
| Velocity accuracy | ±10% | Compare to chronograph |
| Direction accuracy | ±15° | Known-angle projectile |
| False positive rate | < 1 per hour | Monitor during quiet periods |
| Miss rate | < 5% | Known projectile test |

### 14.4 Detection Range Testing

**Range testing procedure:**

1. Set up test range with known projectile velocities
2. Position detection system at increasing distances
3. Record detection success rate vs distance
4. Plot detection probability curve
5. Identify maximum reliable detection distance

**Expected results:**

| Projectile | CW Radar (Standard) | CW Radar (Range-Optimized) |
|------------|---------------------|---------------------------|
| Rifle 850 m/s | 80-100 m | 130-150 m |
| Handgun 350 m/s | 40-60 m | 70-90 m |

---

## 15. Integration with 360° Pendant

### 15.1 Event-Enhanced 360° Pendant Configuration

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

# Haptic for local alert
haptic_type = "lra"
piezo_haptic = false

# Detection configuration
[extreme_velocity.pendant_360]
use_event_cameras = true
trigger_conventional_cameras = true
conventional_camera_capture_s = 2.0
```

### 15.2 Pendant Variant F — Long-Range Detection

**Configuration for maximum detection range:**

```toml
[nodes.pendant_long_range]
# Specialized for long-range rifle detection
variant = "f_long_range"

# Higher-gain antenna
antenna_type = "directional_patch"
antenna_gain_dbi = 15

# CW radar optimized for range
cw_radar_config = "range_optimized"

# Event cameras for confirmation
event_camera_count = 2

# Accept tighter field of view
field_of_view_deg = 60          # Narrower for gain

# Detection range target
target_detection_range_m = 150
```

---

## 16. Safety and Ethical Considerations

### 16.1 Fundamental Physics Constraint

**Detection does NOT prevent impact.**

At close range (< 5 m for rifles, < 3 m for handguns), the warning time is physically insufficient for deliberate protective action. The system provides:
- Situational awareness
- Reflex preparation
- Evidence capture

It does NOT provide:
- Impact prevention
- Evade capability at close range
- Substitute for protective equipment

### 16.2 Realistic Warning Time Analysis

**Rifle at 100 meters:**
- Flight time: 117.6 ms
- Detection + alert: 0.65-2.4 ms (optimized)
- Warning time: 115-117 ms
- Human reaction time: ~200 ms
- **Assessment:** Warning received, startle response possible

**Rifle at 50 meters:**
- Flight time: 58.8 ms
- Detection + alert: 0.65-2.4 ms (optimized)
- Warning time: 56-58 ms
- Human reaction time: ~200 ms
- **Assessment:** Warning received, but limited time for response

**Handgun at 10 meters:**
- Flight time: 28.6 ms
- Detection + alert: 0.65-2.4 ms (optimized)
- Warning time: 26-28 ms
- Human reaction time: ~200 ms
- **Assessment:** Warning received, awareness only

**Handgun at 5 meters:**
- Flight time: 14.3 ms
- Detection + alert: 0.65-2.4 ms (optimized)
- Warning time: 12-14 ms
- Human reaction time: ~200 ms
- **Assessment:** Awareness only; no time for deliberate response

### 16.3 What the System Actually Provides

| Capability | Close-Range (< 10 m) | Medium-Range (10-50 m) | Long-Range (50+ m) |
|------------|---------------------|------------------------|-------------------|
| Extreme velocity detection | ✅ Yes | ✅ Yes | ✅ Yes |
| Direction awareness | ✅ Yes | ✅ Yes | ✅ Yes |
| Alert to wearer | ✅ Yes | ✅ Yes | ✅ Yes |
| Evidence capture | ✅ Yes | ✅ Yes | ✅ Yes |
| Time for response | ❌ No | ⚠️ Limited | ✅ Yes |
| Impact prevention | ❌ No | ❌ No | ❌ No |

### 16.4 Disclaimers

- **Not a certified safety system**
- **Not personal protective equipment**
- **Not a substitute for body armor**
- **Detection capability does not imply protective capability**
- **User assumes all responsibility for deployment**
- **Close-range scenarios are physically constrained — no detection system can provide meaningful warning time at point-blank range**

### 16.5 Recommended User Posture

**For close-range environments:**
- System provides awareness and evidence
- Do not rely on warning time for response
- Combine with protective equipment if threat is anticipated

**For medium/long-range environments:**
- System provides meaningful warning time
- Directional alerts indicate threat vector
- Time available for protective movement or seek cover

**Evidence capture:**
- Works at all distances
- Provides forensic value regardless of warning time
- Integrity chain for legal use

---

## 17. Summary

### 17.1 Key Conclusions

1. **Realistic engagement distances provide substantial warning time:**
   - Rifle at 100 m: 115-117 ms warning
   - Rifle at 200 m: 232-235 ms warning
   - Handgun at 10 m: 26-28 ms warning

2. **Detection range is the primary optimization target:**
   - Maximizing detection distance increases warning time
   - Latency optimization is secondary at realistic rifle distances
   - Range-optimized configuration trades resolution for distance

3. **Tiered architecture separates concerns:**
   - Tier 1 (Reflex): 50-150 µs — trivial compared to flight time
   - Tier 2 (Direction): 100-500 µs — still trivial
   - Tier 3 (Characterization): 5-50 ms — completes well before impact at 100+ m

4. **CW radar is the primary trigger:**
   - No chirp latency
   - Pure Doppler detection
   - 50-150 µs response time

5. **Projectile Doppler signatures are massive:**
   - Rifle: 340 kHz (vs 400 Hz for walking)
   - Trivial to detect with short FFT windows
   - 32 µs window sufficient for threshold detection

6. **Event cameras are faster than originally estimated:**
   - 10-100 µs latency (not 1 ms)
   - Asynchronous event emission
   - Perfect for direction validation

7. **Haptic selection depends on scenario:**
   - LRA sufficient for rifle at 50+ m
   - Piezo valuable for close-range handgun (< 10 m)
   - Piezo not mandatory for typical rifle engagements

8. **BLE optimization is beneficial but not mandatory:**
   - Standard BLE sufficient for rifle at 50+ m
   - Optimized BLE valuable for handgun at < 15 m
   - Adaptive mode provides best of both

### 17.2 Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| CW radar for Tier 1 | Zero chirp latency, pure Doppler, fastest response |
| Event camera for Tier 2 | Microsecond latency, direction extraction |
| FMCW for Tier 3 only | Range + velocity, not on critical path |
| LRA default haptic | Sufficient for majority of realistic scenarios |
| Piezo optional | Close-range enhancement, not mandatory |
| Adaptive BLE | Standard for far, optimized for close |

### 17.3 Final Posture

**The system provides:**
- Excellent detection at realistic distances
- Meaningful warning for rifle at 50+ m
- Awareness for handgun at 10+ m
- Evidence capture at all distances
- Directional alerting for response

**The system does NOT provide:**
- Impact prevention
- Bullet dodging
- Substitute for protective equipment

**The honest truth:**
Detection provides awareness and evidence. At realistic rifle distances (50-300 m), the physics provides adequate warning time. At close range (< 10 m), the physics is constrained and warning time is limited. The system is honest about what it can and cannot do.
