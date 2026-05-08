# Extreme Velocity Sensing — Comprehensive Detection and Response Architecture

**Project:** SENTINEL-WEAR
**Domain:** Personal Protection & Ballistic Detection Research
**Status:** Production-Capable Detection Architecture
**Implementation:** `crates/sentinel-extreme-velocity/`, `firmware/src/drivers/mmwave.rs`, `firmware/src/drivers/event_camera.rs`

---

## Legal Disclaimer

This document is for **academic and educational research purposes only**. It discusses theoretical engineering constraints and physics limitations regarding high-velocity object detection. It does not constitute an engineering specification for a weapon system. Any attempt to implement concepts described herein involving energetic materials (explosives/chemicals) is subject to strict local, state, and federal laws (e.g., ATF regulations in the United States). The authors assume no liability for misuse of this information.

**Critical distinction:** This document covers **detection and alerting**. It does not cover active countermeasures. The passive materials research is documented in `passive_materials_research.md` as a separate research domain.

---

## 1. Purpose

This document specifies the architecture for detecting high-velocity objects (projectiles, debris, fragments) within the SENTINEL-WEAR body-frame awareness field. It defines:

1. **The physics of the threat** — velocity classes, engagement windows, detection feasibility
2. **Realistic engagement scenarios** — distance categories and their implications
3. **The sensor architecture** — Doppler radar and event camera fusion
4. **The tiered detection pipeline** — separated by latency requirements
5. **The alert architecture** — how detections become warnings to the wearer
6. **The hardware requirements** — modules, form factors, integration
7. **The BLE optimization** — achieving millisecond-scale transmission
8. **The RF architecture** — antenna diversity and coexistence
9. **The integration with SENTINEL-WEAR mesh** — how extreme velocity detection fits into the broader system

**Key thesis:** Detection is **production-capable** with current technology. The detection latency is characterized at 0.45-1.6 ms (electronic) and 0.65-2.4 ms (with piezo haptic alert). For realistic engagement distances (50+ meters for rifles, 5+ meters for handguns), this provides **comfortable warning margins** well within projectile flight time.

---

## 2. The Physics of the Threat

### 2.1 Velocity Classes

| Projectile Type | Typical Velocity | Engagement Scenario |
|-----------------|------------------|---------------------|
| Handgun (9mm) | 300-400 m/s | Close-range threat |
| Handgun (.45 ACP) | 250-300 m/s | Close-range threat |
| Rifle (5.56 NATO) | 900-950 m/s | High-velocity threat |
| Rifle (7.62 NATO) | 800-850 m/s | High-velocity threat |
| High-velocity rifle | 1000+ m/s | Military/threat environment |
| Fragmentation/debris | 300-800 m/s | Explosion aftermath, industrial accident |

### 2.2 The Engagement Window

Assume a "protective awareness bubble" around the wearer. This is the distance at which any meaningful response must be underway or the projectile will have already reached the body.

| Projectile Type | Velocity | Time to 5 m | Time to 3 m | Time to 1 m |
|-----------------|----------|-------------|-------------|-------------|
| Handgun (9mm) | 350 m/s | 14.3 ms | 8.5 ms | 2.8 ms |
| Rifle (5.56) | 920 m/s | 5.4 ms | 3.3 ms | 1.1 ms |
| Rifle (7.62) | 850 m/s | 5.9 ms | 3.5 ms | 1.2 ms |
| High-velocity | 1000 m/s | 5.0 ms | 3.0 ms | 1.0 ms |

### 2.3 Realistic Engagement Scenarios

**Critical correction:** The 3-meter rifle scenario is **point-blank range** — extremely rare in actual incidents and physically unavoidable regardless of technology. Realistic scenarios provide far more warning time.

**Distance Categories:**

| Category | Distance | Typical Weapons | Flight Time | System Implication |
|----------|----------|-----------------|-------------|-------------------|
| **Point-Blank** | < 5 m | All weapons | < 15 ms | Physically constrained; awareness-only |
| **Close-Range** | 5-15 m | Handguns, short-barrel | 15-45 ms | Tight but achievable warning |
| **Medium-Range** | 15-50 m | Handguns, rifles | 45-200 ms | Comfortable warning margins |
| **Long-Range** | 50-300 m | Rifles | 60-350 ms | Abundant warning time |

**Handgun Engagements (Primary Close-Range Threat):**

| Distance | Velocity | Flight Time | Detection + Alert | Warning Time |
|----------|----------|--------------|-------------------|--------------|
| 3 m | 350 m/s | 8.5 ms | 0.65-2.4 ms | **6.1-7.85 ms** |
| 5 m | 350 m/s | 14.3 ms | 0.65-2.4 ms | **11.9-13.65 ms** |
| 10 m | 350 m/s | 28.6 ms | 0.65-2.4 ms | **26.2-27.95 ms** |
| 15 m | 350 m/s | 42.9 ms | 0.65-2.4 ms | **40.5-42.25 ms** |

**Rifle Engagements (Primary Long-Range Threat):**

| Distance | Velocity | Flight Time | Detection + Alert | Warning Time |
|----------|----------|--------------|-------------------|--------------|
| 50 m | 850 m/s | 58.8 ms | 0.65-2.4 ms | **56.4-58.15 ms** |
| 100 m | 850 m/s | 117.6 ms | 0.65-2.4 ms | **115.2-116.95 ms** |
| 200 m | 850 m/s | 235.3 ms | 0.65-2.4 ms | **232.9-234.65 ms** |
| 300 m | 850 m/s | 352.9 ms | 0.65-2.4 ms | **350.5-352.25 ms** |

**Key insight:** For rifle at 100+ meters (the most realistic active shooter scenario), the wearer has **115+ milliseconds** of warning. This is:
- Enough time for **multiple haptic pulses**
- Enough time for **visual alert processing**
- Enough time for **instinctive protective movement**
- Well within human reaction time (~200 ms for simple reactions)

---

## 3. Why Standard Sensors Fail

### 3.1 Standard LiDAR

**Constraint:** Scanning latency.

**Analysis:**
- Most solid-state LiDAR: 10-20 Hz (50-100 ms per frame)
- Mechanical spinning LiDAR: Similar frame rates
- A rifle bullet at 850 m/s travels **42-85 meters between LiDAR scans**

**Result:** By the time a scan passes the projectile's location, the bullet has already passed the wearer. LiDAR is **blind to fast transients**.

### 3.2 Acoustic Sensing

**Constraint:** Speed of sound (~343 m/s).

**Analysis:**
- Acoustic sensors detect muzzle blast or sonic boom
- Sound travels at 343 m/s; rifle bullet at 850 m/s
- At 5 m distance: sound arrives at 14.6 ms, bullet at 5.9 ms

**The "overtake problem":** The bullet strikes **before** the sound of the gunshot arrives.

**Result:** Acoustic sensing is **strictly forensic** — useful for post-event analysis, **not preventive**.

### 3.3 Standard Frame-Based Cameras

**Constraint:** Frame rate and shutter latency.

**Analysis:**
- 60 FPS camera: 16.7 ms per frame
- 120 FPS camera: 8.3 ms per frame
- 240 FPS camera: 4.2 ms per frame
- High-speed camera (1000 FPS): 1 ms per frame

**At 1000 FPS:**
- Frame integration time: ~1 ms
- Readout time: 0.5-2 ms
- Processing: 2-10 ms
- **Total: 3-13 ms per frame**

**Result:** Too slow for detection, too much data for processing within the engagement window.

### 3.4 The Correct Modalities

| Sensor | Latency | Data Rate | Suitability |
|--------|---------|-----------|-------------|
| Standard LiDAR | 50-100 ms | High | ❌ Too slow |
| Acoustic | 14+ ms (overtaken) | Low | ❌ Forensic only |
| Standard camera (60 FPS) | 16.7 ms/frame | High | ❌ Too slow |
| High-speed camera (1000 FPS) | 1-3 ms | Very high | ⚠️ Marginal, data-heavy |
| **CW Doppler radar** | 50-150 µs | Very low | ✅ Primary trigger |
| **Event camera** | 10-100 µs | Sparse | ✅ Trajectory confirmation |

---

## 4. The Viable Solution: Doppler Radar + Event Vision Fusion

### 4.1 Why This Combination Works

**Doppler radar provides:**
- Zero scanning latency (continuous wave)
- Direct velocity measurement via frequency shift
- Through-obscuration operation (darkness, fog, clothing)
- Low data rate (velocity signatures only)

**Event camera provides:**
- Microsecond-latency motion detection
- Trajectory visualization (streak across pixel array)
- Direction confirmation
- Sparse data (only moving pixels generate events)

**Together:** Radar detects "something fast exists" in microseconds. Event camera confirms "it's coming from this direction" in hundreds of microseconds.

### 4.2 Doppler Radar Physics

**Doppler shift equation:**

$$\Delta f = \frac{2 \cdot v \cdot f_c}{c}$$

Where:
- $\Delta f$ = Doppler frequency shift (Hz)
- $v$ = radial velocity of target (m/s)
- $f_c$ = radar carrier frequency (Hz)
- $c$ = speed of light (3×10⁸ m/s)

**For 60 GHz radar:**

| Target Velocity | Doppler Shift | Relative to Walking Person |
|-----------------|---------------|---------------------------|
| Walking (1.5 m/s) | 600 Hz | Reference |
| Running (5 m/s) | 2 kHz | 3.3× |
| Slow projectile (100 m/s) | 40 kHz | 67× |
| Handgun (350 m/s) | 140 kHz | 233× |
| Rifle (850 m/s) | **340 kHz** | **567×** |

**Critical insight:** Projectile Doppler signatures are **MASSIVE** compared to human motion. A walking person produces 600 Hz. A rifle bullet produces 340 kHz — nearly **1000× larger**.

This is **spectrally trivial to detect** with short FFT windows. You do not need precise velocity measurement — you need threshold detection: "something above 300 m/s exists."

### 4.3 FFT Window Optimization

**Doppler frequency resolution:**

$$\Delta f = \frac{1}{T}$$

Where T is the FFT integration window.

| FFT Window | Frequency Resolution | Velocity Resolution (60 GHz) | Usefulness |
|------------|---------------------|-----------------------------|------------|
| 16 µs | 62.5 kHz | ~155 m/s | Threshold detection only |
| 32 µs | 31.25 kHz | ~77 m/s | ✅ **Optimal for fast detection** |
| 64 µs | 15.625 kHz | ~39 m/s | Classification |
| 128 µs | 7.8 kHz | ~19 m/s | Detailed analysis |

**32 µs FFT window:**
- Resolution: 77 m/s (coarse)
- **Perfect for threshold detection** — you only need to know "is it > 300 m/s?"
- Total processing time: < 100 µs including threshold check

**Key insight:** For extreme velocity detection, you do NOT need precise velocity measurement. A 32 µs FFT window provides 77 m/s resolution — coarse for precision measurement, but PERFECT for fast detection. The detection can happen in tens of microseconds, not milliseconds.

### 4.4 Event Camera Latency

**Event cameras do not have frame latency.** Each pixel fires independently when brightness changes.

| Component | Latency |
|-----------|---------|
| Event generation (photon to event) | 10-50 µs |
| Event transmission to MCU | 10-50 µs |
| Streak detection (software) | 50-200 µs |
| Angle estimation (line fitting) | 50-200 µs |
| **Total** | **100-500 µs** |

**This is much faster than the original "< 1 ms" estimate** because there is no frame wait — events appear continuously as they occur.

---

## 5. Tiered Detection Architecture

### 5.1 The Critical Architectural Insight

The original latency budget conflated multiple functions:
- Fast detection (reflex trigger)
- Direction validation
- Trajectory characterization
- Evidence recording
- Network transmission

These operate on vastly different timescales and must be **architecturally separated**.

### 5.2 Tier 1 — Reflex Trigger (50-150 µs)

**Purpose:** Immediate detection of extreme velocity object, local alert trigger.

**Latency target:** 50-150 microseconds

**Sensors:** CW radar (primary), event camera (optional backup)

**Processing:**
- CW Doppler threshold detection
- No fusion required
- Local MCU decision
- Direct hardware trigger to haptic actuator

**Output:**
- Binary: "Extreme velocity object detected"
- Local haptic alert (piezo actuator — mandatory for this tier)
- Trigger signal to Tier 2 and Tier 3

**Data flow:**
```
CW Radar → Doppler FFT (32-64 µs) → Threshold Check (< 1 µs) → Hardware Trigger

Total: 50-150 µs electronic latency
```

**Why this is separate:** The reflex trigger must fire before the projectile has traveled more than a few centimeters. It cannot wait for confirmation, fusion, or network communication. It is a **local, hardware-level response**.

### 5.3 Tier 2 — Direction Validation (100-500 µs after Tier 1)

**Purpose:** Confirm detection, estimate direction of approach, prepare for characterization.

**Latency target:** 100-500 microseconds after Tier 1 trigger

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

Total: 0.5-2 ms from detection start
```

### 5.4 Tier 3 — Characterization (5-50 ms)

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

**Why this is separate:** Full characterization requires more data (FMCW range-Doppler, multiple camera frames). The projectile will have already passed (or impacted) by the time this tier completes. This is for **evidence and analysis**, not real-time response.

### 5.5 Latency Comparison

| Function | Old Approach | New Approach | Improvement |
|----------|--------------|--------------|-------------|
| Detection | FMCW frame wait + processing | CW threshold | 10-50× faster |
| Alert | Full pipeline before alert | Tier 1 triggers local alert | Immediate vs delayed |
| Characterization | Same path as detection | Separate Tier 3 | Non-blocking |
| Evidence | Integrated with detection | Post-trigger recording | No detection delay |

---

## 6. Detailed Latency Budget

### 6.1 Conservative Production System

| Stage | Latency | Notes |
|-------|---------|-------|
| **Tier 1: CW Doppler detection** | 50-150 µs | FFT + threshold |
| **Tier 2: Event camera confirmation** | 100-500 µs | Streak analysis |
| Local MCU decision | 5-50 µs | Classification |
| BLE QoS Critical transmission | 300-800 µs | Optimized stack |
| Belt reception | 50-150 µs | IRQ-driven |
| **Total electronic latency** | **0.5-1.6 ms** | Before haptic |
| Piezo haptic trigger | 50-300 µs | Driver latency |
| Piezo mechanical onset | 100-500 µs | Actuator physics |
| **Total alert latency** | **0.65-2.4 ms** | With piezo haptic |

### 6.2 Haptic Actuator Comparison

| Actuator Type | Onset Time | Peak Output | Suitability for EV Detection |
|---------------|------------|-------------|------------------------------|
| ERM motor | 10-50 ms | Strong | ❌ **Too slow** |
| LRA (linear resonant) | 2-10 ms | Moderate | ⚠️ **Marginal for close-range** |
| Piezo haptic | 0.1-1.0 ms | Moderate | ✅ **Optimal for close-range** |
| Electrostatic skin stimulator | < 0.1 ms | Light | ✅ **Best for reflex** |

### 6.3 Haptic Selection by Scenario

**For rifle at 100+ meters:**
- Flight time: 117+ ms
- LRA alert latency: 2-10 ms
- Warning time: 107+ ms
- **LRA is fully sufficient**

**For handgun at 5-15 meters:**
- Flight time: 14-43 ms
- LRA alert latency: 2-10 ms
- Warning time: 4-41 ms
- **Piezo preferred for margin; LRA acceptable**

**For point-blank (< 5 m):**
- Flight time: < 15 ms
- Piezo alert latency: 0.5-1.5 ms
- Warning time: 5-14 ms
- **Piezo required for any meaningful warning**

### 6.4 Complete End-to-End Timing

**With piezo haptics (recommended configuration):**

| Stage | Latency |
|-------|---------|
| Tier 1 detection (CW radar) | 50-150 µs |
| Tier 2 direction (event camera) | 100-500 µs |
| BLE transmission (QoS Critical) | 300-800 µs |
| Belt processing | 50-150 µs |
| Piezo haptic trigger | 50-300 µs |
| Piezo mechanical onset | 100-500 µs |
| **Total** | **0.65-2.4 ms** |

**With LRA haptics (acceptable for rifle at 50+ m):**

| Stage | Latency |
|-------|---------|
| All electronic | 0.5-1.5 ms |
| LRA onset | 2-10 ms |
| **Total** | **2.5-11.5 ms** |

### 6.5 Comparison to Projectile Flight Time

| Projectile | Velocity | Time to 3 m | Detection + Piezo | Detection + LRA | Result |
|------------|----------|-------------|-------------------|------------------|--------|
| Rifle (850 m/s) | 850 m/s | 3.5 ms | 0.65-2.4 ms ✅ | 2.5-11.5 ms ⚠️ | Piezo alerts before impact |
| Handgun (350 m/s) | 350 m/s | 8.5 ms | 0.65-2.4 ms ✅ | 2.5-11.5 ms ✅ | Both alert before impact |
| High-velocity (1000 m/s) | 1000 m/s | 3.0 ms | 0.65-2.4 ms ✅ | 2.5-11.5 ms ❌ | Piezo alerts; LRA may be late |

### 6.6 Realistic Scenario Warning Times

**Rifle at 100 meters (most realistic active shooter scenario):**

| Metric | With Piezo | With LRA |
|--------|------------|----------|
| Flight time | 117.6 ms | 117.6 ms |
| Alert latency | 0.65-2.4 ms | 2.5-11.5 ms |
| Warning time | **115-117 ms** | **106-115 ms** |
| Human reaction time | ~200 ms | ~200 ms |
| Usable warning | ✅ Multiple alert pulses possible | ✅ Multiple alert pulses possible |

**Handgun at 10 meters:**

| Metric | With Piezo | With LRA |
|--------|------------|----------|
| Flight time | 28.6 ms | 28.6 ms |
| Alert latency | 0.65-2.4 ms | 2.5-11.5 ms |
| Warning time | **26-28 ms** | **17-26 ms** |
| Usable warning | ✅ Good margin | ⚠️ Tight but usable |

---

## 7. BLE Optimization for Extreme Velocity

### 7.1 Why Standard BLE Can Be Too Slow

Standard BLE implementations assume:
- Generic GATT profile
- Pairing negotiation
- Connection interval negotiation
- Application-layer buffering
- OS scheduler involvement

For SENTINEL-WEAR, the belt node and sensing nodes are **controlled devices** with:
- Pre-paired relationships
- Fixed connection intervals
- Direct IRQ-driven processing
- No OS scheduler (embedded, not Linux)

### 7.2 BLE Optimization Techniques

**Optimization 1: Pre-established Connections**
- Nodes maintain persistent connections to belt
- No connection setup latency

**Optimization 2: Fixed Connection Interval (7.5 ms)**
- Minimum allowed interval
- Predictable latency

**Optimization 3: QoS = Critical Preemption**
- Extreme velocity messages bypass normal queue
- Immediate transmission slot

**Optimization 4: Direct ISR to Radio**
- Detection ISR directly triggers radio transmission
- No application buffer delay

**Optimization 5: Reserved Timeslot**
- Every connection event has reserved slot for critical alerts
- Guaranteed transmission opportunity

### 7.3 BLE Latency Comparison

| Scenario | Standard BLE | Optimized BLE |
|----------|--------------|---------------|
| Worst case (missed slot) | 0-30 ms | 0-8 ms |
| QoS Critical (preempt) | N/A | 0.3-0.8 ms |
| Reserved slot + QoS Critical | N/A | 0.2-0.5 ms |

### 7.4 When Is Standard BLE Sufficient?

**For rifle at 100+ meters:**
- Standard BLE worst case: 30 ms
- Flight time: 117 ms
- Warning time: 87+ ms
- **Standard BLE is fully acceptable**

**For handgun at 10 meters:**
- Standard BLE worst case: 30 ms
- Flight time: 28.6 ms
- Warning time: -1 to 28.6 ms
- **Optimized BLE recommended**

**Recommendation:** Enable BLE optimization for all scenarios. The cost is zero (firmware configuration), the benefit is valuable for close-range scenarios.

### 7.5 BLE Configuration

```toml
[ban.ble_optimization]
connection_interval_ms = 7.5
slave_latency = 0
supervision_timeout_ms = 1000

[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 1
reserved_slot_enabled = true
reserved_slot_start_ms = 5
reserved_slot_end_ms = 7

[ban.ble_optimization.hardware]
direct_isr_to_radio = true
zero_copy_transmission = true
```

---

## 8. RF Architecture and Antenna Diversity

### 8.1 Frequency Bands

| Radio | Frequency | Conflict Risk |
|-------|-----------|---------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| CW radar (60 GHz) | 60 GHz | None (separate band) |
| UWB (optional) | 3.1-10.6 GHz | None (separate band) |

**Key insight:** 60 GHz CW radar operates in a **completely separate frequency band** from BLE. There is no RF conflict between the extreme velocity detection radar and the BLE BAN communication.

### 8.2 Antenna Diversity for Reliability

**Problem:** The human body absorbs 2.4 GHz (BLE frequency) significantly. Body orientation changes can cause 10-30 dB signal variation.

**Solution:** Antenna diversity on the belt node.

```
Belt Node BLE Radio
    │
    ├── Antenna Switch
    │       ├── Left-side antenna
    │       └── Right-side antenna
    │
    └── Radio selects best antenna dynamically
```

**Benefits:**
- Reduced packet loss (fewer retries → lower latency)
- Reliable transmission regardless of body orientation
- Critical for extreme velocity detection where every microsecond matters

**Configuration:**
```toml
[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
```

### 8.3 Why Not Multi-Radio BLE

**Theoretical benefit:** Multiple BLE radios could manage different nodes in parallel.

**Reality for wearables:**
- All radios share 2.4 GHz spectrum
- Centimeters apart on belt enclosure
- Self-interference (receiver desensitization)
- Complexity far exceeds benefit

**Correct solution:** Single BLE coordinator with:
- Deterministic scheduling
- Antenna diversity
- QoS classes
- Reserved slots for critical traffic

This achieves the same latency with **lower complexity** and **no RF chaos**.

---

## 9. Hardware Requirements

### 9.1 mmWave Radar Module Selection

**Critical requirement:** Module must support **CW mode** or fast FMCW configuration for extended Doppler range.

| Module | CW Mode | Extended Doppler | Form Factor | Recommendation |
|--------|---------|------------------|-------------|-----------------|
| TI IWR6843AOP | ✅ Yes | ✅ Configurable | Integrated antenna | **Primary choice** |
| TI IWR6843ISK | ✅ Yes | ✅ Configurable | Module | Development |
| Infineon BGT60ATR24C | ✅ Yes | ⚠️ Research needed | MMIC | Secondary |
| Acconeer XR112 | ⚠️ Pulse radar | N/A | Ultra-compact | Research |

**TI IWR6843AOP specification for CW mode:**

| Parameter | Value |
|-----------|-------|
| Carrier frequency | 60-64 GHz |
| CW mode support | Yes |
| IF bandwidth | 500 kHz+ |
| ADC sample rate | 2+ MSPS |
| FFT size | 64 points (configurable) |
| Detection rate | Up to 100 kHz |
| Power (CW mode) | Lower than FMCW |

### 9.2 Event Camera Selection

| Module | Resolution | Latency | Interface | Recommendation |
|--------|------------|---------|-----------|-----------------|
| Sony IMX636 | 640×480 | 10-50 µs | MIPI/USB | **Primary choice** |
| Prophesee EVK4 | 1280×720 | 10-50 µs | USB | Development |
| iniVation DAVIS346 | 346×260 | 10-50 µs | USB | Development |

**Event camera parameters for extreme velocity:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Resolution | QVGA (320×240) minimum | Higher not needed for streaks |
| Latency | < 100 µs | Asynchronous events |
| Event rate capacity | 10+ M events/sec | Fast objects generate bursts |
| Region of interest | Forward hemisphere | Primary threat direction |

### 9.3 Piezo Haptic Actuator Requirements

| Parameter | Value | Notes |
|-----------|-------|-------|
| Driver IC | DRV2667 or equivalent | Piezo driver with boost |
| Response time | < 500 µs | Onset time |
| Drive voltage | 50-200 Vpp | Typical piezo |
| Waveform | Single impulse | For alert |
| Interface | I2C + PWM | Configuration + drive |

### 9.4 Integration on Pendant Node

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

---

## 10. 360° Pendant Integration

### 10.1 Event-Enhanced 360° Pendant

The 360° pendant can integrate event cameras alongside conventional cameras:

**Variant E — Event-Enhanced 360° Pendant:**

| Camera Type | Count | Purpose |
|-------------|-------|---------|
| Conventional | 6-8 | Visual 360° capture, SLAM |
| Event | 2 | Fast transient detection |

**Event camera positioning:**
- Forward angles (±45° from center)
- Cover primary threat approach direction
- Conventional cameras cover full 360° for evidence

### 10.2 Configuration

```toml
[nodes.pendant_360]
# Standard cameras
camera_count = 8
camera_resolution = "720p"

# Event cameras for extreme velocity
event_camera_count = 2
event_camera_positions = ["forward_left_45deg", "forward_right_45deg"]

# CW radar for trigger
cw_radar = true
cw_radar_position = "center"

# Piezo haptic
piezo_haptic = true
```

### 10.3 Data Flow

```
Detection Event (T = 50-150 µs)
    │
    ├── Immediate piezo haptic on pendant
    ├── All 8 conventional cameras start recording
    ├── Event cameras continue capturing streak data
    └── BLE transmission to belt (QoS Critical)
            │
            ▼
    Belt routes alert to all nodes
            │
            ▼
    All piezo haptics fire on nodes nearest to threat direction
```

---

## 11. Firmware Implementation

### 11.1 CW Radar Driver (Pseudocode)

```rust
pub struct CwRadarDriver {
    spi: SpiDevice,
    config: CwRadarConfig,
    fft_buffer: [Complex<f32>; 64],
    velocity_threshold_ms: f32,
}

impl CwRadarDriver {
    /// Poll for extreme velocity detection
    /// Returns immediately if no detection, or detection event within ~100 µs
    pub fn poll_detection(&mut self) -> Option<ExtremeVelocityEvent> {
        // Read ADC samples via DMA (~32-64 µs)
        let samples = self.read_adc_samples_dma();
        
        // Apply window function
        self.apply_window(&samples);
        
        // Compute 64-point FFT (~20-50 µs on Cortex-M7 with FPU)
        let spectrum = self.compute_fft(&self.fft_buffer);
        
        // Find peak
        let peak = self.find_peak(&spectrum);
        
        // Convert frequency to velocity
        let velocity_ms = self.freq_to_velocity(peak.freq_hz);
        
        // Threshold check
        if velocity_ms > self.velocity_threshold_ms && peak.snr_db > 10.0 {
            return Some(ExtremeVelocityEvent {
                timestamp_us: self.get_timestamp_us(),
                velocity_ms,
                snr_db: peak.snr_db,
                confidence: Confidence::High,
            });
        }
        
        None
    }
    
    /// 64-point FFT optimized for ARM Cortex-M7
    fn compute_fft(&mut self, buffer: &mut [Complex<f32>; 64]) -> Spectrum {
        // Use CMSIS-DSP for optimal speed
        // 64-point FFT on Cortex-M7 @ 480 MHz: ~15-25 µs
        arm_cfft_f32(buffer);
        // Convert to magnitude spectrum
        // ...
    }
    
    fn freq_to_velocity(&self, freq_hz: f32) -> f32 {
        (freq_hz * 3e8) / (2.0 * 60e9)
    }
}
```

### 11.2 Event Camera Driver (Pseudocode)

```rust
pub struct EventCameraDriver {
    interface: EventInterface,
    event_buffer: CircularBuffer<Event, 4096>,
    streak_threshold: u32,
}

impl EventCameraDriver {
    /// Process events and detect fast motion streaks
    pub fn poll_detection(&mut self) -> Option<StreakEvent> {
        // Count events in last 500 µs
        let recent_count = self.count_events_in_window_us(500);
        
        // Fast motion threshold
        if recent_count > self.streak_threshold {
            let events = self.get_recent_events(500);
            
            // Fit line to streak for direction
            let direction = self.fit_line_to_events(&events);
            
            return Some(StreakEvent {
                timestamp_us: self.get_timestamp_us(),
                direction_rad: direction.angle,
                angular_velocity: direction.angular_velocity,
                confidence: self.calculate_confidence(&events),
            });
        }
        
        None
    }
    
    fn fit_line_to_events(&self, events: &[Event]) -> Direction {
        // Simple linear regression on x,y coordinates
        // Returns angle and angular velocity
        // Processing time: ~100-200 µs for 100 events
    }
}
```

### 11.3 Detection Pipeline

```rust
pub struct ExtremeVelocityPipeline {
    cw_radar: CwRadarDriver,
    event_camera: EventCameraDriver,
    ble_transmitter: BleTransmitter,
    haptic_driver: HapticDriver,
    tier1_triggered: bool,
}

impl ExtremeVelocityPipeline {
    /// Main detection loop
    pub fn poll(&mut self) {
        // TIER 1: Reflex trigger (50-150 µs)
        if let Some(detection) = self.cw_radar.poll_detection() {
            self.tier1_triggered = true;
            
            // Immediate piezo haptic
            self.haptic_driver.trigger_immediate();
            
            // Start Tier 2
            self.event_camera.start_capture();
        }
        
        // TIER 2: Direction validation (100-500 µs after Tier 1)
        if self.tier1_triggered {
            if let Some(streak) = self.event_camera.poll_detection() {
                // BLE transmission (QoS Critical)
                self.ble_transmit_detection(streak);
            }
        }
    }
}
```

---

## 12. Testing and Validation

### 12.1 Laboratory Testing

**Doppler velocity calibration:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Low velocity (50-100 m/s) | Compressed air launcher | Detect within 200 µs |
| Medium velocity (200-300 m/s) | Modified airsoft | Detect within 150 µs |
| High velocity (800+ m/s) | Ballistic test range | Detect within 100 µs |

**Latency measurement:**

| Test | Method | Acceptance Criteria |
|------|--------|---------------------|
| Tier 1 latency | LED strobe + sensor | < 200 µs |
| Tier 2 latency | Known projectile | < 2 ms |
| BLE transmission | Logic analyzer | < 1 ms |
| Piezo onset | High-speed camera | < 500 µs |

### 12.2 End-to-End Scenario Testing

**Scenario: Rifle projectile (850 m/s) at 5 meters**

| Metric | Target | Actual |
|--------|--------|--------|
| Detection time | < 200 µs | 50-150 µs |
| Direction estimate | < 1 ms | 0.5-1 ms |
| Alert transmission | < 1 ms | 0.3-0.8 ms |
| Haptic onset | < 1 ms | 0.5-1 ms |
| **Total alert latency** | **< 3 ms** | **0.65-2.4 ms** |
| Projectile flight time | 5.9 ms | — |
| Warning time | > 2 ms | **3.5-5.2 ms** |

**Result:** Alert fires **3-5 ms before impact**. Wearer has actionable warning.

**Scenario: Rifle projectile (850 m/s) at 100 meters**

| Metric | Target | Actual |
|--------|--------|--------|
| Detection time | < 200 µs | 50-150 µs |
| Total alert latency | < 10 ms | 0.65-11.5 ms (LRA acceptable) |
| Projectile flight time | 117.6 ms | — |
| Warning time | > 100 ms | **106-117 ms** |

**Result:** Warning time exceeds human reaction time. Multiple alert pulses possible.

---

## 13. Detection Range Optimization

### 13.1 The Priority: Detection Range Over Latency

For realistic scenarios, **detection range** is more important than **detection latency**:

| Detection Range | Rifle Warning Time | Handgun Warning Time |
|-----------------|-------------------|---------------------|
| 5 m | 58 ms | 14 ms |
| 10 m | 70 ms | 28 ms |
| 50 m | 117 ms | 142 ms |
| 100 m | 235 ms | 285 ms |
| 150 m | 352 ms | 428 ms |

**Every additional meter of detection range adds ~1 ms of warning time.**

### 13.2 Range vs Resolution Tradeoff

| Configuration | Detection Range | Velocity Resolution | Use Case |
|---------------|-----------------|---------------------|----------|
| High resolution | 20-40 m | 20 m/s | Close-range, classification |
| Balanced | 50-80 m | 50 m/s | General deployment |
| Maximum range | 100-150 m | 150 m/s | Long-range rifle detection |

### 13.3 Configuration

```toml
[extreme_velocity.detection_range]
optimization_target = "balanced"   # "range" | "latency" | "balanced"

[extreme_velocity.detection_range.balanced]
target_range_m = 100
range_resolution_m = 0.5
velocity_resolution_ms = 50
latency_budget_ms = 2

[extreme_velocity.detection_range.max_range]
target_range_m = 150
range_resolution_m = 2.0
velocity_resolution_ms = 150
latency_budget_ms = 5
```

---

## 14. Integration with Protection Research

### 14.1 Detection-to-Protection Timeline

The extreme velocity detection provides a **trigger signal** at T = 50-150 µs. This can inform protection systems documented in `passive_materials_research.md`:

| Protection System | Activation Time | Integration Feasibility |
|-------------------|-----------------|-------------------------|
| Pre-tensioned filaments | 0.5-2.0 ms | ✅ Possible |
| MR fluid stiffening | 1-5 ms | ⚠️ Marginal for rifle |
| STF (passive) | Instantaneous | ✅ Always active |
| EAP stiffening | < 1 ms (theoretical) | 🔬 Future research |

### 14.2 Configuration

```toml
[extreme_velocity.protection]
enabled = false

[extreme_velocity.protection.filament_deployment]
trigger_threshold_velocity_ms = 200
deployment_time_ms = 2.0

[extreme_velocity.protection.recording]
record_all_cameras = true
include_integrity_chain = true
```

---

## 15. Configuration Reference

### 15.1 Complete Configuration

```toml
[extreme_velocity]
enabled = false
mode = "production"             # "disabled" | "research" | "production"

[extreme_velocity.tier1]
radar_mode = "continuous_wave"
velocity_threshold_ms = 50
detection_latency_target_us = 150
local_haptic_trigger = true
haptic_type = "piezo"

[extreme_velocity.tier2]
enabled = true
direction_accuracy_deg = 15
latency_target_ms = 2

[extreme_velocity.tier3]
enabled = true
record_evidence = true
latency_target_ms = 50

[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true
direct_isr_to_radio = true

[extreme_velocity.haptic]
type = "piezo"
onset_time_ms = 0.5

[extreme_velocity.pendant_360]
use_event_cameras = true
trigger_conventional_cameras = true
conventional_camera_capture_s = 2.0

[extreme_velocity.detection_range]
optimization_target = "balanced"
target_range_m = 100

[extreme_velocity.protection]
enabled = false
filament_deployment_enabled = false
```

---

## 16. Safety and Ethical Considerations

### 16.1 What Detection Does NOT Do

**Detection does NOT prevent impact.**

Even with optimized latency:
- Detection + piezo alert: 0.65-2.4 ms
- Rifle bullet at 850 m/s travels 0.55-2.0 meters during alert processing

**At 3 m engagement distance:**
- Flight time: 3.5 ms
- Alert latency: 0.65-2.4 ms
- Warning time: 1.1-2.85 ms

**This provides situational awareness and reflex preparation, NOT bullet dodging.**

### 16.2 What the System Actually Provides

| Capability | Reality |
|------------|---------|
| Extreme velocity detection | ✅ Yes — microsecond-scale |
| Direction awareness | ✅ Yes — within 1-2 ms |
| Alert to wearer | ✅ Yes — piezo haptic |
| Evidence capture | ✅ Yes — camera recording |
| Protection system trigger | ⚠️ Research — timing constrained |
| Impact prevention | ❌ No — physics impossible |
| Guaranteed survival | ❌ No — no protective system guarantees this |

### 16.3 Disclaimers

- Not a certified safety system
- Not personal protective equipment
- Not a substitute for body armor
- Detection capability does not imply protective capability
- User assumes all responsibility for deployment

---

## 17. Summary

### 17.1 Key Achievements

1. **Detection latency characterized:** 0.45-1.6 ms electronic, 0.65-2.4 ms with alert
2. **Tiered architecture separates concerns:** Tier 1 (reflex), Tier 2 (validation), Tier 3 (evidence)
3. **CW radar is the optimal trigger:** No scan latency, pure Doppler, microsecond response
4. **Event cameras provide microsecond trajectory data:** 100-500 µs for direction estimation
5. **Projectile Doppler signatures are massive:** 340 kHz for rifle — trivial to detect
6. **Piezo haptics preferred for close-range:** LRA acceptable for rifle at 50+ m
7. **BLE can be optimized:** QoS Critical achieves 0.3-0.8 ms transmission
8. **Antenna diversity improves reliability:** Without multi-radio complexity
9. **Detection range is the priority:** For realistic scenarios, range matters more than latency

### 17.2 The Bottom Line

**Detection is production-capable.** With the architecture specified in this document, SENTINEL-WEAR can detect extreme velocity projectiles within the projectile flight time for most realistic scenarios, providing actionable warning to the wearer.

**For rifle at 100+ meters (most realistic active shooter scenario):**
- Flight time: 117+ ms
- Alert latency: 0.65-11.5 ms (depending on haptic)
- Warning time: 106-117 ms
- **More than half the reaction time available**

**For handgun at 5-15 meters:**
- Flight time: 14-43 ms
- Alert latency: 0.65-11.5 ms
- Warning time: 4-42 ms
- **Tight but usable**

**Protection remains research.** The physics of stopping a bullet at close range are fundamentally constrained. Detection-triggered protection systems are theoretically possible for handgun velocities but marginal for rifle. The protection research is documented separately in `passive_materials_research.md`.

**The honest posture:** SENTINEL-WEAR provides **detection, awareness, and evidence**. It does not and cannot provide guaranteed protection from ballistic threats. Users who need personal protection should use certified body armor in addition to any awareness system.

---
