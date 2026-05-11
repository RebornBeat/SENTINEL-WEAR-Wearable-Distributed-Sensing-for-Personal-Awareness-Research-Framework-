# RF Coexistence Architecture — SENTINEL-WEAR

**Project:** SENTINEL-WEAR
**Domain:** Radio frequency coexistence, interference mitigation, antenna architecture
**Status:** Reference Design
**Version:** 1.0

---

## 1. Overview

SENTINEL-WEAR operates multiple radio systems simultaneously in close physical proximity on the human body. This document specifies the RF coexistence architecture, interference mitigation strategies, and antenna design principles that ensure reliable operation of all wireless subsystems.

### 1.1 Design Philosophy

**Core principle:** RF coexistence is achieved through architectural separation, not brute-force isolation. The system is designed so that each radio operates in its natural domain without fighting for spectrum.

**Key architectural decisions:**
- Single BLE coordinator (not multiple BLE radios) on belt node
- Spectral separation between primary subsystems
- Antenna diversity for reliability (not multiple simultaneous transmitters)
- Deterministic scheduling to prevent contention
- External connectivity concentrated at belt node only

---

## 2. Frequency Bands in Use

### 2.1 Complete Radio Inventory

| Radio System | Frequency Band | Nodes Present | Modulation | Primary Purpose |
|--------------|---------------|---------------|------------|-----------------|
| **BLE 5.x** | 2.400–2.485 GHz (2.4 GHz ISM) | All nodes | GFSK, π/4-DQPSK, 8DPSK | BAN communication |
| **WiFi** | 2.400–2.485 GHz (2.4 GHz ISM) | Belt only | OFDM | Companion app connection |
| **WiFi** | 5.150–5.875 GHz (5 GHz U-NII) | Belt only | OFDM | Companion app (preferred) |
| **UWB** | 3.1–10.6 GHz | Optional on any node | IR-UWB | Precision timing, ranging, bandwidth augmentation |
| **mmWave Radar** | 57.0–64.0 GHz (60 GHz band) | Pendant, bracelets, belt | FMCW / CW | Presence, velocity, geometry sensing |
| **Cellular LTE** | 700 MHz – 2.6 GHz | Belt only | OFDMA / SC-FDMA | Remote access, alerts |
| **Cellular 5G Sub-6** | 3.3–4.2 GHz | Belt only | CP-OFDM | High-bandwidth remote |
| **Cellular 5G mmWave** | 24.25–52.6 GHz | Belt only | CP-OFDM | Maximum bandwidth remote |

### 2.2 Conflict Matrix

| Radio Pair | Frequency Relationship | Conflict Risk | Mitigation Required |
|------------|----------------------|---------------|---------------------|
| BLE ↔ WiFi 2.4 GHz | **Same band** | **HIGH** | Required (see Section 4) |
| BLE ↔ WiFi 5 GHz | Separate bands | None | None |
| BLE ↔ UWB | Separate bands | Minor | None |
| BLE ↔ mmWave | Far separated | None | None |
| BLE ↔ Cellular LTE | Adjacent (LTE Band 7) | Minor | Minimal |
| BLE ↔ Cellular 5G Sub-6 | Adjacent to UWB | Minor | Minimal |
| WiFi 2.4 GHz ↔ WiFi 5 GHz | Same device, separate bands | Internal coordination | AP selection |
| WiFi ↔ mmWave | Far separated | None | None |
| UWB ↔ mmWave | Separate bands | None | None |
| UWB ↔ Cellular 5G Sub-6 | Adjacent | Minor | Minimal |
| mmWave ↔ Cellular 5G mmWave | Adjacent (60 vs 24-47 GHz) | Minor | Minimal |
| Cellular LTE ↔ Cellular 5G | Separate bands | Internal coordination | Module design |

### 2.3 Key Insight: The Only Real Conflict

**The only significant RF conflict in SENTINEL-WEAR is BLE ↔ WiFi at 2.4 GHz.**

All other radio pairs are either:
- In completely separate frequency bands (mmWave, UWB)
- On the same device with internal coordination (WiFi bands, cellular bands)
- Far enough apart that natural isolation is sufficient

**This simplifies coexistence engineering dramatically.**

---

## 3. Why Multi-Radio BLE Is NOT the Solution

### 3.1 The Attractive Premise

A naive approach to BAN bandwidth might suggest:

```
Multiple BLE radios on belt node:
├── BLE Radio #1 → Pendant
├── BLE Radio #2 → Anklet Left
├── BLE Radio #3 → Anklet Right
├── BLE Radio #4 → Bracelet Left
└── BLE Radio #5 → Bracelet Right
```

**Theoretical benefits:**
- Parallel connections instead of sequential
- Lower per-node latency
- More aggregate bandwidth
- Reduced scheduling complexity

### 3.2 Why This Fails in Wearable Form Factor

#### 3.2.1 Self-Interference Physics

All BLE radios would share:
- Same 2.4 GHz ISM band
- Same 40 channels
- Same adaptive frequency hopping algorithm
- Centimeters of physical separation
- Same ground plane
- Same power supply
- Same enclosure
- Proximity to human body (high RF absorption)

**Result: Receiver desensitization**

```
BLE Radio A transmits at 0 dBm
    ↓ (signal couples to nearby antenna)
BLE Radio B's receiver front-end partially saturates
    ↓ (sensitivity degrades by 10-20 dB)
BLE Radio B misses packets
    ↓ (retransmissions required)
Overall latency INCREASES instead of decreasing
```

This is a well-documented phenomenon in dense embedded RF systems.

#### 3.2.2 Required Mitigations (If Multi-Radio Attempted)

| Mitigation | Complexity | Effectiveness | Cost Impact |
|------------|------------|---------------|-------------|
| Antenna isolation (>20 dB) | High PCB complexity | Moderate | +30% PCB cost |
| RF shielding cans | Adds height, cost | Good | +$5-10 per unit |
| Guard traces on PCB | Moderate | Moderate | Minor |
| Power supply isolation | High complexity | Good | +20% power circuit cost |
| Synchronized frequency hopping | Firmware complexity | High | Development time |
| Cross-radio coordination firmware | Very high | Essential | Significant development |

**Net result:** Complexity increases dramatically for marginal or negative gains.

#### 3.2.3 Why Single-Radio BLE Is Sufficient

**BLE bandwidth is NOT saturated by SENTINEL-WEAR payloads.**

| Traffic Type | Payload Size | Frequency | Bandwidth Required |
|--------------|--------------|-----------|-------------------|
| IMU orientation | 16 bytes | 50-200 Hz | ~25 KBps |
| Detection event | 32-64 bytes | Event-driven | <1 KBps |
| Gait event | 48 bytes | 1-2 Hz | <1 KBps |
| Classification result | 16-32 bytes | Event-driven | <1 KBps |
| Health telemetry | 32 bytes | 0.1 Hz | <1 KBps |

**Total sustained:** < 50 KBps = 400 Kbps

**BLE 5.x practical bandwidth:** 500 Kbps - 2 Mbps

**The BAN uses < 10% of BLE's capacity.** Adding radios would not improve a system that is not bandwidth-constrained.

### 3.3 When Multi-Radio BLE IS Appropriate

Multi-radio BLE architecture makes sense for:
- Industrial sensor mesh (dozens of nodes, room-scale)
- Hospital patient monitoring (ward-scale)
- Smart building deployment (floor-wide)
- Multi-user environment (conference center)

**NOT appropriate for:** Single-person wearable system with 4-6 nodes.

---

## 4. The Correct Solution: Antenna Diversity

### 4.1 Single Radio + Multiple Antennas

**Antenna diversity uses ONE BLE radio with multiple antenna paths:**

```
Belt Node BLE Radio
    │
    ├── Antenna Switch (SPDT)
    │       ├── Antenna #1 (left side of belt)
    │       └── Antenna #2 (right side of belt)
    │
    └── Radio dynamically selects best antenna
```

**Key difference from multi-radio:**
- ONE transmitter active at any time
- ONE receiver active at any time
- NO self-interference
- Antenna selection takes microseconds

### 4.2 Why Antenna Diversity Is Critical for Wearables

#### 4.2.1 Body Shadowing Problem

The human body is a significant RF absorber at 2.4 GHz:

| Body Part | Approximate Attenuation | Notes |
|-----------|------------------------|-------|
| Torso | 10-30 dB | Depends on body composition |
| Arm | 5-15 dB | Varies with arm position |
| Head | 15-25 dB | Significant blockage |
| Leg | 10-20 dB | When leg is between Tx and Rx |

**Single antenna problem:**

```
Node on wearer's right side
    ↓ (body blocks direct path)
Single antenna on left side of belt
    ↓ (signal must diffract around body)
Path loss increases 20-30 dB
    ↓
Packet loss or retries
```

**Dual antenna solution:**

```
Node on wearer's right side
    ↓
Left antenna: poor signal (-80 dBm)
Right antenna: good signal (-60 dBm)
    ↓
Radio switches to right antenna
    ↓
Reliable communication
```

#### 4.2.2 Antenna Diversity Benefits

| Benefit | Impact | Quantification |
|---------|--------|----------------|
| Reduced packet loss | Fewer retries | 50-80% reduction |
| More stable latency | Deterministic timing | ±5 ms vs ±20 ms |
| Body orientation independence | Reliable in any pose | Works while turning, sitting, lying |
| Improved range | Better link budget | +10-20 dB effective gain |

### 4.3 Antenna Diversity Implementation

#### 4.3.1 Hardware Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BELT NODE ANTENNA DIVERSITY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────┐                                                          │
│   │ BLE Radio    │                                                          │
│   │ (nRF5340)    │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│          │ RF Output                                                         │
│          ▼                                                                   │
│   ┌──────────────┐                                                          │
│   │ SPDT RF      │                                                          │
│   │ Switch       │◄── GPIO Select from MCU                                  │
│   │ (SKY13330)   │                                                          │
│   └──────┬───────┘                                                          │
│          │                                                                   │
│    ┌─────┴─────┐                                                            │
│    │           │                                                            │
│    ▼           ▼                                                            │
│ ┌────────┐ ┌────────┐                                                       │
│ │Antenna │ │Antenna │                                                       │
│ │#1      │ │#2      │                                                       │
│ │(Left)  │ │(Right) │                                                       │
│ └────────┘ └────────┘                                                       │
│                                                                              │
│ Physical separation: 80-120 mm across belt enclosure                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 Antenna Switch Specifications

| Parameter | Typical Value | Notes |
|-----------|---------------|-------|
| Switching time | < 1 µs | Fast enough for packet-by-packet selection |
| Insertion loss | 0.5-1.0 dB | Minor impact on link budget |
| Isolation | 20-30 dB | Between antenna paths |
| Control interface | GPIO | Simple MCU control |
| Power consumption | < 1 µA | Negligible |

#### 4.3.3 Antenna Types

| Antenna Type | Advantages | Disadvantages | Recommendation |
|--------------|------------|---------------|----------------|
| PCB trace (inverted-F) | Zero cost, integrated | Fixed orientation | Good for mass production |
| Chip antenna | Small, omnidirectional | Requires ground plane | Good for compact designs |
| Flex antenna | Conformal to enclosure | Higher cost | Best for curved belt enclosures |

### 4.4 Antenna Selection Algorithm

#### 4.4.1 RSSI-Based Selection

```rust
/// Antenna diversity selection algorithm
pub struct AntennaDiversityController {
    current_antenna: Antenna,
    rssi_history: [RssiSample; 16],
    switch_threshold_db: f32,
    hysteresis_db: f32,
    min_switch_interval_ms: u32,
    last_switch_time: u64,
}

impl AntennaDiversityController {
    /// Call after each received packet
    pub fn update(&mut self, packet_rssi: i8, current_time: u64) {
        // Record RSSI for current antenna
        self.record_rssi(self.current_antenna, packet_rssi);
        
        // Check if we should consider switching
        if current_time - self.last_switch_time < self.min_switch_interval_ms as u64 {
            return; // Too soon since last switch
        }
        
        // Compare average RSSI between antennas
        let current_avg = self.average_rssi(self.current_antenna);
        let other_avg = self.average_rssi(self.current_antenna.opposite());
        
        // Switch if other antenna is significantly better
        if other_avg > current_avg + self.switch_threshold_db + self.hysteresis_db {
            self.switch_antenna(current_time);
        }
    }
    
    fn switch_antenna(&mut self, current_time: u64) {
        self.current_antenna = self.current_antenna.opposite();
        self.last_switch_time = current_time;
        
        // Configure RF switch GPIO
        unsafe { RF_SWITCH_GPIO.write_volatile(self.current_antenna.gpio_value()); }
    }
}
```

#### 4.4.2 Packet-Loss Triggered Selection

```rust
/// Alternative: Switch on packet loss detection
impl AntennaDiversityController {
    pub fn on_packet_loss(&mut self, current_time: u64) {
        // Immediate switch on packet loss
        // Body shadowing likely; try other antenna
        if current_time - self.last_switch_time > self.min_switch_interval_ms as u64 {
            self.switch_antenna(current_time);
        }
    }
}
```

#### 4.4.3 Hybrid Algorithm (Recommended)

```rust
/// Best practice: Use both RSSI and packet-loss triggers
impl AntennaDiversityController {
    pub fn update_hybrid(&mut self, event: RadioEvent, current_time: u64) {
        match event {
            RadioEvent::PacketReceived { rssi } => {
                self.record_rssi(self.current_antenna, rssi);
                self.evaluate_switch(current_time);
            }
            RadioEvent::PacketLost => {
                self.on_packet_loss(current_time);
            }
            RadioEvent::TransmissionStarting => {
                // Pre-select best antenna based on recent history
                self.preselect_for_transmit();
            }
        }
    }
}
```

### 4.5 Configuration

```toml
[ban.antenna_diversity]
# Master enable
enabled = true

# Hardware configuration
antenna_count = 2                    # 1 = single, 2 = diversity
switch_gpio_left = "PA0"             # GPIO for left antenna selection
switch_gpio_right = "PA1"            # GPIO for right antenna selection

# Algorithm parameters
selection_mode = "hybrid"            # "rssi" | "packet_loss" | "hybrid"
switch_threshold_db = 5.0            # Switch if other antenna is 5 dB better
hysteresis_db = 2.0                  # Prevent rapid switching
min_switch_interval_ms = 50          # Don't switch more often than this

# RSSI averaging window
rssi_window_size = 16                # Number of samples to average
rssi_weight_recent = 0.3             # Weight for most recent samples

# Performance targets
target_packet_loss_percent = 1.0     # Acceptable packet loss rate
target_latency_stability_ms = 5      # Target latency standard deviation
```

---

## 5. BLE ↔ WiFi Coexistence

### 5.1 The Core Conflict

Both BLE and WiFi operate in the 2.4 GHz ISM band:

| Protocol | Channels | Bandwidth per Channel | Frequency Range |
|----------|----------|----------------------|-----------------|
| BLE | 40 channels (3 advertising) | 2 MHz | 2400-2483.5 MHz |
| WiFi 2.4 GHz | 11-14 channels (by region) | 20-22 MHz | 2412-2472 MHz |

**WiFi channel overlaps multiple BLE channels.** When WiFi transmits, nearby BLE channels experience interference.

### 5.2 Strategy 1: WiFi Prefers 5 GHz Band (Primary Mitigation)

**This is the primary and most effective mitigation.**

**Implementation:**
- Belt node WiFi radio configured to prefer 5 GHz networks
- Most modern routers support 5 GHz
- 5 GHz provides faster throughput anyway

**Configuration:**
```toml
[rf_coexistence]
wifi_prefer_5ghz = true              # Strongly recommended
wifi_5ghz_ssid_preference = []       # Specific SSIDs to prefer, empty = any 5 GHz
wifi_fallback_to_24ghz = true        # Fall back if 5 GHz unavailable
```

**Result:** BLE operates at 2.4 GHz; WiFi operates at 5 GHz. **No conflict.**

### 5.3 Strategy 2: Antenna Separation

When 2.4 GHz WiFi must be used:

```
Belt Node Enclosure (Top View)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   [Cellular Antenna]                    [WiFi Antenna]                      │
│   (Exterior, toward outside)            (Exterior, toward outside)          │
│   ← Maximum separation from BLE →                                       │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │              PCB / Internal Electronics                     │          │
│   │                                                             │          │
│   │   [BLE Antenna #1]           [BLE Antenna #2]               │          │
│   │   (Interior, toward body)    (Interior, toward body)        │          │
│   │                                                             │          │
│   └─────────────────────────────────────────────────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Principle:**
- WiFi antennas on exterior of belt (better propagation, away from body)
- BLE antennas on interior (body is the BAN propagation medium)
- Physical separation between WiFi and BLE antennas reduces coupling

### 5.4 Strategy 3: Time-Division Multiplexing

When both 2.4 GHz WiFi and BLE must transmit simultaneously:

```toml
[rf_coexistence.ble_wifi_timeshare]
enabled = false                     # Enable only if 2.4 GHz WiFi unavoidable
ble_gap_during_wifi_ms = 5          # BLE quiet period during WiFi transmission
wifi_burst_before_ble = true        # Complete WiFi burst before resuming BLE
```

**Implementation:**
```
Time Division Schedule (if timeshare enabled):

|←──── WiFi Active ────→|← BLE Quiet →|←──── BLE Active ────→|
                        │             │
                        WiFi completes│BLE resumes
                        transmission  │
```

**This requires firmware coordination between WiFi and BLE stacks.**

### 5.5 Strategy 4: BLE Frequency Hopping Awareness

BLE uses adaptive frequency hopping (AFH). The radio automatically avoids channels with interference.

**Enhancement:** Firmware can explicitly mark WiFi-occupied channels as "bad" to accelerate AFH:

```rust
/// Mark WiFi channel as interfered for BLE AFH
pub fn configure_ble_afh_for_wifi(&mut self, wifi_channel: u8) {
    // WiFi channel 1 (2412 MHz) overlaps BLE channels 0-8
    // WiFi channel 6 (2437 MHz) overlaps BLE channels 13-21
    // WiFi channel 11 (2462 MHz) overlaps BLE channels 26-34
    
    let ble_channels_to_avoid = wifi_channel_to_ble_channels(wifi_channel);
    
    for ch in ble_channels_to_avoid {
        self.ble_radio.mark_channel_unusable(ch);
    }
}
```

### 5.6 Combined Coexistence Configuration

```toml
[rf_coexistence]
# Primary strategy
wifi_prefer_5ghz = true              # Use 5 GHz WiFi (eliminates conflict)

# Fallback if 2.4 GHz WiFi required
wifi_fallback_to_24ghz = true

# Secondary strategies
[rf_coexistence.ble_wifi_timeshare]
enabled = false                     # Only if 2.4 GHz WiFi active
ble_gap_during_wifi_ms = 5

[rf_coexistence.afh_enhancement]
mark_wifi_channels = true           # Tell BLE which channels WiFi uses
update_on_wifi_channel_change = true

# Antenna configuration (always active)
[rf_coexistence.antenna_separation]
ble_antennas_interior = true        # BLE antennas on body side
wifi_antenna_exterior = true        # WiFi antenna on outside
```

---

## 6. BLE Scheduling and Determinism

### 6.1 The Scheduling Problem

If all nodes transmit simultaneously over a single BLE radio:
- Packets collide
- Retries increase
- Latency becomes unpredictable
- Bandwidth is wasted

### 6.2 Solution: Time-Slotted Scheduling

**Belt node as coordinator assigns specific time slots to each node:**

```
BLE Connection Event (30 ms interval, total frame)
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
├── Slot 25-28 ms: Reserved (critical alerts, extreme velocity)
└── Slot 28-30 ms: Retransmits / contention window
```

**Benefits:**
- Each node has guaranteed transmission opportunity
- No packet collisions within the mesh
- Deterministic latency bounds
- Predictable power consumption

### 6.3 Connection Interval Tradeoffs

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture, extreme velocity |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

### 6.4 Dynamic Scheduling

**Context-aware adjustment:**

| Context Detected | Schedule Adjustment |
|------------------|---------------------|
| Walking | Prioritize anklets (gait timing critical) |
| Threat detected | Prioritize pendant, reserved slot |
| Idle | Extend intervals, reduce power |
| Streaming active | Shift heavy traffic to UWB |
| Extreme velocity | Preempt all, use reserved slot |

### 6.5 QoS Classes

| QoS Class | Priority | Preemption | Use Cases |
|-----------|----------|------------|-----------|
| **Critical** | Highest | Yes, immediate | Extreme velocity, fall, emergency |
| **High** | Second | No | Gait anomaly, stumble precursor |
| **Medium** | Third | No | Presence detection, tracking |
| **Low** | Fourth | No | Battery status, telemetry |

### 6.6 Configuration

```toml
[ban.scheduling]
mode = "dynamic"                    # "static" | "dynamic"

[ban.scheduling.static]
frame_interval_ms = 30
slot_allocation = "default"

[ban.scheduling.dynamic]
# Context-aware adjustments
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0
streaming_shift_to_uwb = true

# Reserved slot for critical alerts
reserved_slot_enabled = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28
reserved_slot_qos = ["critical"]

# QoS configuration
[ban.qos]
classes = ["critical", "high", "medium", "low"]

[ban.qos.critical]
preempt_other_traffic = true
immediate_transmit = true
max_latency_ms = 5

[ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20

[ban.qos.medium]
max_latency_ms = 50

[ban.qos.low]
max_latency_ms = 500
```

---

## 7. UWB Coexistence

### 7.1 UWB Frequency Position

UWB operates in **3.1-10.6 GHz**, which is:
- Completely separate from 2.4 GHz (BLE, WiFi)
- Separate from 5 GHz WiFi
- Separate from 60 GHz mmWave
- Adjacent to but separate from 5G Sub-6 GHz cellular

**UWB has NO significant conflict with BLE, WiFi, or mmWave.**

### 7.2 UWB ↔ 5G Sub-6 GHz

**Minor consideration:** If belt node has 5G cellular operating in 3.5 GHz band, this is adjacent to UWB.

**Mitigation:**
- Physical antenna separation
- UWB and 5G typically not active simultaneously (UWB for BAN, 5G for remote access)
- Filter design at antenna input

**Typical isolation:** 30+ dB with proper antenna placement

### 7.3 UWB and BLE Co-Operation

UWB and BLE are **complementary**, not competing:

| Transport | Role | When Active |
|-----------|------|-------------|
| BLE | Control, metadata, health | Always |
| UWB | Timing, ranging, bandwidth burst | When enabled and needed |

**Operating together:**
```
BLE: Handles connection establishment, health monitoring, low-bandwidth sensor data
UWB: Provides precision timing, ranging, and high-bandwidth bursts
```

### 7.4 UWB Configuration

```toml
[ban.uwb]
enabled = false                     # Opt-in per deployment
role = "timing_and_ranging"         # "timing_only" | "timing_and_ranging" | "full_bandwidth"

[ban.uwb.coexistence]
# UWB operates in separate band from BLE - no direct conflict
# Coordinate operation for power management
pause_during_cellular_5g_sub6 = true  # Minor precaution
coordinate_with_ble_timing = true     # BLE connection events as UWB timing reference
```

---

## 8. Cellular Coexistence

### 8.1 Cellular Bands and Potential Conflicts

| Cellular Band | Frequency | Conflict Risk |
|---------------|-----------|---------------|
| LTE Band 1 | 2100 MHz | None (far from BLE) |
| LTE Band 3 | 1800 MHz | None (far from BLE) |
| LTE Band 7 | 2600 MHz | Minor (adjacent to BLE band) |
| LTE Band 20 | 800 MHz | None |
| LTE Band 28 | 700 MHz | None |
| 5G n78 | 3.5 GHz | Minor (adjacent to UWB) |
| 5G mmWave | 24-47 GHz | None (far from all) |

### 8.2 LTE Band 7 (2.6 GHz) — The Only Concern

**LTE Band 7 uplink (2500-2570 MHz) is adjacent to BLE upper band (2483.5 MHz).**

**Mitigation:**
- Cellular antenna on belt exterior
- BLE antennas on belt interior
- 20-30 dB isolation through physical separation
- BLE AFH avoids affected channels if needed

**In practice:** The 15 MHz gap between BLE (2483.5 MHz max) and LTE B7 (2500 MHz min) provides sufficient isolation for typical wearable power levels.

### 8.3 5G Sub-6 GHz and UWB

**Both operate near 3-4 GHz range.**

**Mitigation:**
- Time-domain separation: 5G used for remote streaming; UWB used for local BAN
- Antenna separation
- Band-pass filtering

### 8.4 Cellular Configuration

```toml
[connectivity.cellular]
enabled = false

[connectivity.cellular.coexistence]
# Cellular antenna placement
antenna_placement = "exterior"       # On outside of belt
antenna_separation_from_ble_mm = 50  # mm from BLE antennas

# Frequency band preferences (if carrier supports multiple)
prefer_bands = [1, 3, 20, 28]        # Avoid Band 7 if possible
avoid_bands = [7]                    # Adjacent to BLE

# 5G Sub-6 and UWB
[connectivity.cellular.5g_sub6]
pause_uwb_during_transmit = true     # Conservative precaution
uwb_resume_delay_ms = 10
```

---

## 9. mmWave Radar (60 GHz) — No Conflict

### 9.1 Frequency Position

mmWave radar operates at **57-64 GHz**, which is:
- 23× higher frequency than 2.4 GHz BLE
- 12× higher than 5 GHz WiFi
- 6× higher than UWB
- 1.5× higher than 5G mmWave

**mmWave radar has NO conflict with any other radio in the system.**

### 9.2 Coexistence Notes

**Interference with 5G mmWave (24-47 GHz):**
- 13-37 GHz frequency gap
- No direct interference
- Antenna design is completely different (arrays vs single element)

**Self-interference with other mmWave nodes:**
- If multiple nodes have mmWave (pendant + bracelets)
- Standard practice: Time-division multiplexing or code-division multiplexing
- Coordinated by belt node scheduling

### 9.3 mmWave Radar Configuration

```toml
[sensors.mmwave_radar]
frequency_ghz = 60

[sensors.mmwave_radar.coexistence]
# No frequency conflicts with BLE/WiFi/Cellular
# Coordinate operation if multiple mmWave nodes present
multi_node_coordination = "tdm"      # "tdm" = time-division multiplexing
interference_detection = true
```

---

## 10. Antenna Placement Strategy

### 10.1 Belt Node Antenna Layout

```
Belt Node Enclosure (Top View - Worn at waist)

                    Front of Wearer
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                                              ┌───┐                          │
│                              [WiFi Antenna]  │   │                          │
│                                    5 GHz     │   │                          │
│                                              └───┘                          │
│                                                    │                         │
│   ┌───┐                                           │                         │
│   │   │  [Cellular Antenna]                       │                         │
│   │   │  (LTE/5G, exterior)                       │                         │
│   └───┘                                         │                         │
│     Left                                        │                          │
│     Side                                        │                          │
│       │                                         │                          │
│       │  ┌─────────────────────────────────────────────────────────┐        │
│       │  │                    PCB / Electronics                    │        │
│       │  │                                                         │        │
│       │  │   [BLE Ant. #1]              [BLE Ant. #2]               │        │
│       │  │   (Left interior)           (Right interior)            │        │
│       │  │        ▲                           ▲                    │        │
│       │  │        │      Body Contact         │                    │        │
│       │  │        └───────────────────────────┘                    │        │
│       │  │                                                         │        │
│       │  └─────────────────────────────────────────────────────────┘        │
│       │                                                                    │
│       │                                                                    │
│   ┌───┐                                                                   │
│   │   │  [mmWave Antenna]                                                 │
│   │   │  (60 GHz, separate from others)                                   │
│   └───┘                                                                   │
│                                                                              │
│                                                                              │
│                                                                              │
│                         [Optional: GNSS Antenna]                            │
│                         (if cellular module includes GPS)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                    Rear of Wearer
```

### 10.2 Antenna Separation Guidelines

| Antenna Pair | Minimum Separation | Reason |
|--------------|-------------------|--------|
| BLE ↔ WiFi 2.4 GHz | 50 mm | Same band, reduce coupling |
| BLE ↔ WiFi 5 GHz | 20 mm | Different bands, minimal coupling |
| BLE ↔ Cellular | 40 mm | Prevent desense |
| WiFi ↔ Cellular | 30 mm | Different bands, moderate isolation |
| mmWave ↔ All Others | 10 mm | Different band, no interaction |

### 10.3 Pendant Node Antenna Layout

```
Pendant Node (Worn at chest)

                    Front of Wearer
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   [mmWave Radar Antenna]                                        │
│   (60 GHz, forward-facing)                                      │
│                                                                  │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                    PCB                              │      │
│   │                                                     │      │
│   │   [BLE Antenna]    [BLE Antenna]                    │      │
│   │   (Toward body)    (Toward body, opposite side)     │      │
│   │                                                     │      │
│   │   Diversity optional on pendant                    │      │
│   │   (Body is not as strong a shield from chest)      │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                  │
│   [Optional UWB Antenna]                                        │
│   (For high-bandwidth pendant variant)                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 10.4 Bracelet Node Antenna Layout

```
Bracelet Node (Worn on wrist)

                    Toward Hand
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   [mmWave Radar Antenna]                                        │
│   (60 GHz, outward-facing)                                      │
│                                                                  │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                    PCB                              │      │
│   │                                                     │      │
│   │   [BLE Antenna #1]    [BLE Antenna #2]             │      │
│   │   (Toward body)       (Away from body)             │      │
│   │                                                     │      │
│   │   Diversity is IMPORTANT on bracelet              │      │
│   │   Arm position highly variable                    │      │
│   │   Body shadowing significant                     │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                  │
│   [Haptic Actuator]                                             │
│   (On skin side)                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                    Toward Elbow
```

**Bracelet antenna diversity is highly recommended** because:
- Arm rotates and moves constantly
- Body shadowing changes rapidly
- One antenna may be blocked by arm position

### 10.5 Anklet Node Antenna Layout

```
Anklet Node (Worn on ankle)

                    Toward Foot
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   [ToF/LiDAR Window]                                            │
│   (Forward/downward)                                            │
│                                                                  │
│   ┌─────────────────────────────────────────────────────┐      │
│   │                    PCB                              │      │
│   │                                                     │      │
│   │   [BLE Antenna #1]    [BLE Antenna #2]             │      │
│   │   (Front of ankle)   (Back of ankle)              │      │
│   │                                                     │      │
│   │   Diversity important due to leg movement          │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                  │
│   [Haptic Actuator]                                             │
│   (On skin side)                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                    Toward Knee
```

---

## 11. Shielding and Isolation Techniques

### 11.1 When Shielding Is Needed

| Scenario | Shielding Required | Technique |
|----------|-------------------|-----------|
| Multi-radio on same PCB | Yes | RF cans, grounded shields |
| Sensitive receiver near transmitter | Yes | Shield cans |
| Cellular module on belt | Moderate | Physical separation often sufficient |
| mmWave near other radios | No | Frequency separation sufficient |

### 11.2 Shielding Implementation

**RF Shield Cans:**
- Metal can over sensitive circuitry
- Grounded to PCB ground plane
- Typical attenuation: 20-40 dB

```
┌─────────────────────────────────────────────────────────────────┐
│                    PCB Cross-Section                             │
│                                                                  │
│   ┌─────────────────────────────────────────────────────┐      │
│   │ RF Shield Can (soldered to ground)                 │      │
│   │ ┌─────────────────────────────────────────────────┐│      │
│   │ │ Protected Circuitry (e.g., BLE receiver)       ││      │
│   │ │                                                 ││      │
│   │ └─────────────────────────────────────────────────┘│      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                  │
│   ────────────────────────────────────────────────────────────  │
│                      PCB Ground Plane                            │
│   ────────────────────────────────────────────────────────────  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 11.3 Guard Traces

**For sensitive analog sections:**
- Grounded guard traces around sensitive signals
- Prevents coupling between adjacent traces
- 3W rule: Guard trace 3× signal trace width away

### 11.4 Power Supply Isolation

**Separate power domains for different radios:**

| Power Domain | Supplies | Isolation Method |
|--------------|----------|------------------|
| `V_BLE` | BLE radio | LC filter |
| `V_WIFI` | WiFi module | Separate LDO |
| `V_CELLULAR` | Cellular module | Buck converter with enable |
| `V_SENSORS` | Radar, IMU, etc. | LC filter |

**Configuration:**
```toml
[power.domains]
ble_isolated = true
wifi_isolated = true
cellular_isolated = true
shared_sensor_rail = true
```

---

## 12. Performance Testing

### 12.1 Coexistence Test Scenarios

| Test ID | Description | Pass Criteria |
|---------|-------------|---------------|
| COEX-01 | BLE + WiFi 5 GHz simultaneously | BLE PER < 1%, WiFi throughput > 80% of solo |
| COEX-02 | BLE + WiFi 2.4 GHz simultaneously | BLE PER < 5%, WiFi throughput > 60% of solo |
| COEX-03 | BLE + Cellular active | BLE PER < 2% |
| COEX-04 | BLE + UWB active | BLE PER < 1%, UWB ranging error < 5 cm |
| COEX-05 | All radios active | System remains functional, no lockups |

### 12.2 Antenna Diversity Testing

| Test ID | Description | Pass Criteria |
|---------|-------------|---------------|
| DIV-01 | Single antenna, body shadowing | PER < 30% (baseline) |
| DIV-02 | Dual antenna, body shadowing | PER < 5% (improvement) |
| DIV-03 | Antenna switch time | < 1 ms to complete switch |
| DIV-04 | RSSI accuracy for selection | ±3 dB |

### 12.3 Latency Under Load Testing

| Test ID | Description | Pass Criteria |
|---------|-------------|---------------|
| LAT-01 | BLE latency (idle) | 10-30 ms (standard interval) |
| LAT-02 | BLE latency (WiFi active) | < 40 ms |
| LAT-03 | BLE latency (streaming) | < 30 ms |
| LAT-04 | Extreme velocity QoS Critical | < 5 ms |

### 12.4 Test Equipment

| Equipment | Purpose |
|-----------|---------|
| Vector network analyzer | Antenna impedance, isolation |
| Spectrum analyzer | RF emissions, interference |
| BLE sniffer | Packet capture, timing analysis |
| WiFi analyzer | Throughput, channel utilization |
| Logic analyzer | GPIO timing for antenna switch |
| Anechoic chamber | Isolated RF testing |
| Body phantom | Simulated human body for wear testing |

---

## 13. Debugging RF Issues

### 13.1 Symptom: High BLE Packet Loss

**Diagnostic steps:**
1. Check RSSI from both ends (belt and node)
2. Verify antenna is connected properly
3. Check for WiFi 2.4 GHz activity
4. Test with WiFi disabled
5. Test with antenna diversity enabled/disabled
6. Check body position/orientation

**Common causes:**
- WiFi 2.4 GHz interfering (solution: prefer 5 GHz)
- Antenna orientation wrong for body position (solution: enable diversity)
- Damaged antenna or connector
- Interference from external source (microwave, other WiFi)

### 13.2 Symptom: Latency Spikes

**Diagnostic steps:**
1. Enable BLE latency logging
2. Check for retry bursts in logs
3. Verify scheduling configuration
4. Check for WiFi heavy traffic
5. Verify no multi-node transmission collision

**Common causes:**
- Retries due to interference (solution: improve coexistence)
- WiFi traffic bursts (solution: timeshare or prefer 5 GHz)
- Node transmitting out of scheduled slot (firmware issue)

### 13.3 Symptom: UWB Ranging Errors

**Diagnostic steps:**
1. Verify UWB antenna is connected
2. Check for 5G cellular activity
3. Test in isolated environment
4. Verify time synchronization

**Common causes:**
- Antenna connection issue
- Multi-path in environment (normal, not a bug)
- 5G Sub-6 GHz interference (rare, solution: pause UWB during 5G)

---

## 14. Configuration Summary

### 14.1 Complete RF Coexistence Configuration

```toml
# RF Coexistence Configuration

[rf_coexistence]
# Primary strategy
wifi_prefer_5ghz = true              # Use 5 GHz WiFi (eliminates BLE conflict)
wifi_fallback_to_24ghz = true

# Antenna configuration
[rf_coexistence.antenna_separation]
ble_antennas_interior = true        # BLE antennas toward body
wifi_antenna_exterior = true        # WiFi antenna toward outside
cellular_antenna_exterior = true

# BLE-WiFi timeshare (only if 2.4 GHz WiFi active)
[rf_coexistence.ble_wifi_timeshare]
enabled = false
ble_gap_during_wifi_ms = 5

# AFH enhancement
[rf_coexistence.afh_enhancement]
mark_wifi_channels = true
update_on_wifi_channel_change = true

# Antenna diversity
[ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "hybrid"
switch_threshold_db = 5.0
hysteresis_db = 2.0
min_switch_interval_ms = 50

# BLE scheduling
[ban.scheduling]
mode = "dynamic"

[ban.scheduling.dynamic]
walking_anklet_priority_boost = 1.5
threat_pendant_priority_boost = 2.0
idle_interval_multiplier = 2.0

# Reserved slot for critical
reserved_slot_enabled = true
reserved_slot_start_ms = 25
reserved_slot_end_ms = 28

# QoS
[ban.qos]
classes = ["critical", "high", "medium", "low"]

# UWB coexistence
[ban.uwb.coexistence]
pause_during_cellular_5g_sub6 = true
coordinate_with_ble_timing = true

# Cellular coexistence
[connectivity.cellular.coexistence]
antenna_placement = "exterior"
antenna_separation_from_ble_mm = 50
prefer_bands = [1, 3, 20, 28]
avoid_bands = [7]

# mmWave coordination
[sensors.mmwave_radar.coexistence]
multi_node_coordination = "tdm"
interference_detection = true

# Power domains
[power.domains]
ble_isolated = true
wifi_isolated = true
cellular_isolated = true

# Debug logging
[rf_coexistence.debug]
log_ble_rssi = true
log_wifi_channel_changes = true
log_antenna_switches = true
log_retry_events = true
```

---

## 15. Summary

### 15.1 Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| Single BLE coordinator | Avoids self-interference, sufficient bandwidth |
| WiFi prefers 5 GHz | Eliminates primary RF conflict |
| Antenna diversity | Improves reliability without adding radios |
| Deterministic scheduling | Prevents collisions, bounds latency |
| External connectivity at belt only | Concentrates RF complexity in one location |

### 15.2 Primary Conflict Resolution

| Conflict | Resolution | Effectiveness |
|----------|------------|----------------|
| BLE ↔ WiFi 2.4 GHz | Prefer 5 GHz WiFi | Eliminates conflict |
| BLE ↔ WiFi 2.4 GHz (fallback) | Antenna separation + timeshare | Reduces conflict |
| BLE body shadowing | Antenna diversity | Eliminates shadowing issue |
| UWB ↔ others | Spectral separation | No conflict |
| mmWave ↔ others | Spectral separation | No conflict |
| Cellular ↔ BLE | Antenna separation | Minimal interaction |

### 15.3 The Bottom Line

**SENTINEL-WEAR's RF architecture is correctly designed:**

- Single BLE coordinator with deterministic scheduling
- WiFi at 5 GHz (eliminating the only significant conflict)
- Antenna diversity for reliability
- All other radios in spectrally separated bands

**Multi-radio BLE would add complexity without benefit.**

**Antenna diversity provides the reliability improvement that multi-radio was trying to solve, without the self-interference problems.**

---

**End of Document**
