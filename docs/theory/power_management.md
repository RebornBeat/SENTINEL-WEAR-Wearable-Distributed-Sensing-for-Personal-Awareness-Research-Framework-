# Power Management — SENTINEL-WEAR

**Project:** SENTINEL-WEAR
**Domain:** Wearable power architecture, battery sizing, thermal management, runtime optimization
**Status:** Research Reference Design
**Version:** 1.0

---

## 1. Philosophy: Power as the Primary Constraint

In wearable sensing systems, power is not a secondary consideration — it is the primary constraint that shapes every other decision. Unlike fixed installations where power is effectively unlimited, a wearable system must balance sensing capability against runtime, thermal comfort against processing throughput, and feature richness against battery size and weight.

**Core Principles:**
- **All-day usability is the baseline requirement** — a wearable that lasts 2 hours is not practical for most use cases
- **Thermal comfort is non-negotiable** — skin-contact temperature above 42°C is unacceptable
- **Power-aware features** — the system should degrade gracefully as battery depletes
- **User-configurable tradeoffs** — users choose between capability and runtime

---

## 2. Power Constraints by Node Type

### 2.1 The Hierarchy of Power Consumption

| Node Type | Typical Power Range | Battery Size | Primary Power Consumer |
|-----------|---------------------|--------------|------------------------|
| Bracelet | 10-500 mW | 150-400 mAh | mmWave radar |
| Anklet | 10-300 mW | 300-500 mAh | ToF/LiDAR + IMU |
| Pendant (Standard) | 50-1000 mW | 150-300 mAh | mmWave + acoustic |
| Pendant (360°) | 1.5-4.5 W | 400-1000 mAh | Camera array + vision processor |
| Belt (MCU) | 200-800 mW | 2000 mAh | BLE hub + sparse tracking |
| Belt (Linux SoM) | 0.5-5 W | 5000-7000 mAh | SLAM + streaming + WiFi |

### 2.2 The Fundamental Asymmetry

The belt node has the largest battery (5-7 Ah) and the highest power draw (up to 5 W). The pendant and bracelets have small batteries (0.15-0.5 Ah) and lower power draw (0.05-1 W).

**Implication:** The belt node is the power reservoir. Extended runtime for high-power nodes (like the 360° pendant) should draw from belt power when possible.

---

## 3. Belt Node Power Architecture

### 3.1 Power Domain Breakdown

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BELT NODE POWER DOMAINS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  V_BAT (3.7-4.2V) — Main Battery Bank                                        │
│  ├── V_MCU (3.3V) — MCU core and digital logic                               │
│  ├── V_RADIO (3.3V) — BLE radio and optional UWB                             │
│  ├── V_WIFI (3.3V) — WiFi module (if external)                               │
│  ├── V_CELLULAR (3.3V/4.2V) — Cellular module                                │
│  ├── V_SENSORS (3.3V) — Belt-mounted sensors (mmWave, IMU, env)              │
│  └── V_PERIPHERAL (5V) — USB output, external power to other nodes           │
│                                                                              │
│  Power Consumers:                                                            │
│  ├── MCU Core: 50-150 mW (idle to active)                                    │
│  ├── BLE Radio: 5-50 mW (standby to active TX)                              │
│  ├── UWB Radio: 50-150 mW (when active)                                     │
│  ├── WiFi Radio: 100-500 mW (idle to streaming)                              │
│  ├── Cellular: 150-1200 mW (idle to active streaming)                       │
│  ├── Sensors: 100-300 mW (continuous)                                        │
│  ├── Linux SoM: 500-3000 mW (idle to full load)                             │
│  └── SLAM Processing: 1000-3000 mW (when active)                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Operating Modes and Power Draw

#### MCU-Class Belt (Variant A)

| Mode | Components Active | Power Draw | Runtime (2000 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising only, MCU low-power | 10-30 mW | 60-150 hours |
| **Idle** | BLE connected, sensor idle | 50-100 mW | 20-40 hours |
| **Standard** | BAN hub, sparse tracking, WiFi idle | 200-400 mW | 10-20 hours |
| **Active** | BAN hub, processing, WiFi connected | 400-800 mW | 5-10 hours |

**No SLAM capability.** MCU-class belt nodes cannot run dense SLAM or video streaming.

#### Linux SoM Belt (Variant B)

| Mode | Components Active | Power Draw | Runtime (5000 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE only, Linux suspended | 50-100 mW | 50-100 hours |
| **Idle** | BLE + WiFi idle, sparse tracking | 300-500 mW | 10-17 hours |
| **Standard** | BAN hub, sparse tracking, WiFi ready | 500-1000 mW | 5-10 hours |
| **Active** | Dense SLAM, processing, WiFi connected | 2000-3000 mW | 3-5 hours |
| **Streaming** | SLAM + camera streaming + WiFi TX | 3000-4500 mW | 2-3.5 hours |
| **Remote** | All above + cellular active | 3500-5300 mW | 1.5-3 hours |

#### High-Performance Belt (Variant C)

| Mode | Components Active | Power Draw | Runtime (7000 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE only, NPU idle | 80-150 mW | 45-90 hours |
| **Idle** | BLE + WiFi idle, sparse tracking | 400-600 mW | 12-18 hours |
| **Standard** | BAN hub, sparse tracking | 600-1200 mW | 6-12 hours |
| **Active** | Dense SLAM, NPU inference | 2500-4000 mW | 3-6 hours |
| **Streaming** | SLAM + NPU + streaming | 4000-6000 mW | 2-3.5 hours |
| **Maximum** | SLAM + NPU + streaming + cellular | 5000-7000 mW | 1.5-2.5 hours |

### 3.3 Cellular Power Impact

Cellular is the most power-variable component:

| Cellular State | Power Draw | Notes |
|----------------|------------|-------|
| Module off | 0 mW | No power consumption |
| Module on, idle | 50-100 mW | Registered to network |
| Module on, connected | 150-300 mW | Data connection established |
| Active TX/RX (LTE Cat 1) | 300-600 mW | Low bandwidth |
| Active TX/RX (LTE Cat 4) | 400-800 mW | Moderate bandwidth |
| Active TX/RX (5G) | 800-1200 mW | High bandwidth streaming |

**Impact on runtime:**

| Belt Configuration | Without Cellular | With Cellular (idle) | With Cellular (active) |
|-------------------|------------------|---------------------|------------------------|
| Linux SoM, Standard mode | 5-10 hours | 4.5-9 hours | 3.5-7 hours |
| Linux SoM, Streaming | 2-3.5 hours | 1.8-3.2 hours | 1.3-2.5 hours |

### 3.4 WiFi Power Impact

| WiFi State | Power Draw | Notes |
|------------|------------|-------|
| Radio off | 0 mW | No power consumption |
| Idle, associated | 100-200 mW | Connected to AP |
| Active RX | 200-350 mW | Downloading |
| Active TX (streaming) | 350-500 mW | Uploading video |

---

## 4. 360° Pendant Power Architecture

### 4.1 Power Domain Breakdown

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      360° PENDANT POWER DOMAINS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  V_BAT (3.7-4.2V) — Pendant Battery                                          │
│  ├── V_MCU (3.3V) — Main MCU and digital logic                               │
│  ├── V_CAMS (2.8V/3.3V) — All N camera modules (ganged)                      │
│  ├── V_ISP (1.8V/1.0V) — Vision processor core and IO                        │
│  ├── V_RADAR (3.3V) — mmWave radar                                           │
│  ├── V_IMU (3.3V/1.8V) — IMU                                                 │
│  └── V_RADIO (3.3V) — BLE/UWB radio                                          │
│                                                                              │
│  Optional External Power:                                                    │
│  V_BELT_IN (5V) — From belt node battery (extended runtime)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Power Consumption

| Component | Idle | Active | Peak | Notes |
|-----------|------|--------|------|-------|
| **Camera (each)** | 5-20 mW | 80-200 mW | 250 mW | Depends on resolution, FPS |
| **8 Cameras (total)** | 40-160 mW | 640-1600 mW | 2000 mW | All active simultaneously |
| **Vision Processor** | 50-100 mW | 500-1500 mW | 2000 mW | Stitching + encoding |
| **mmWave Radar** | 100-200 mW | 200-400 mW | 600 mW | Continuous monitoring |
| **IMU** | 1-5 mW | 10-30 mW | 50 mW | 200 Hz sampling |
| **BLE Radio** | 5-20 mW | 50-150 mW | 200 mW | Transmission bursts |
| **UWB Radio** | 20-50 mW | 80-200 mW | 300 mW | If populated and active |

### 4.3 Operating Modes and Power Draw

| Mode | Components Active | Power Draw | Runtime (800 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising only | 10-30 mW | 30-90 hours |
| **Sparse sensing** | Radar + IMU, no cameras | 200-500 mW | 6-15 hours |
| **Cameras low-res** | 8 cameras QVGA, stitching | 1000-2000 mW | 1.5-3 hours |
| **Cameras standard** | 8 cameras VGA, stitching | 2000-3000 mW | 1-1.5 hours |
| **Cameras high-res** | 8 cameras 720p, stitching | 3000-4000 mW | 0.7-1 hour |
| **Full active** | All + UWB streaming | 3500-4500 mW | 0.5-0.85 hours |

### 4.4 Extended Runtime with Belt Power

**Belt power input enables:**
- Unlimited runtime while connected
- Active 360° streaming without battery drain on pendant
- Full capability operation during extended events

**Physical implementation:**
- Conductive traces in necklace chain
- Or: Separate power cable from belt to pendant
- Or: Magnetic connector on pendant back

**Configuration:**
```toml
[nodes.pendant_360.power]
belt_power_input = true           # Enable belt power input
belt_power_threshold_percent = 30 # Switch to belt power when pendant < 30%
priority = "belt_first"           # "belt_first" | "pendant_first" | "auto"
```

---

## 5. Standard Pendant Power Architecture

### 5.1 Component Power Consumption

| Component | Idle | Active | Peak |
|-----------|------|--------|------|
| **mmWave Radar** | 50-100 mW | 150-300 mW | 500 mW |
| **IMU** | 1-5 mW | 10-30 mW | 50 mW |
| **Microphone Array** | 10-30 mW | 50-150 mW | 200 mW |
| **Optional Camera** | 20-50 mW | 150-400 mW | 600 mW |
| **BLE Radio** | 5-20 mW | 50-100 mW | 150 mW |

### 5.2 Operating Modes

| Mode | Components Active | Power Draw | Runtime (250 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising | 10-30 mW | 20-50 hours |
| **Standard** | Radar + IMU + acoustic | 200-500 mW | 2-5 hours |
| **With camera** | All + camera | 400-800 mW | 1-2.5 hours |

---

## 6. Bracelet Node Power Architecture

### 6.1 Component Power Consumption

| Component | Idle | Active | Peak |
|-----------|------|--------|------|
| **mmWave Radar** | 30-80 mW | 100-250 mW | 400 mW |
| **IMU** | 1-5 mW | 5-20 mW | 30 mW |
| **Haptic (LRA)** | 0 mW | 100-300 mW | 500 mW (pulse) |
| **BLE Radio** | 5-15 mW | 30-80 mW | 120 mW |
| **Optional Camera** | 20-40 mW | 100-300 mW | 500 mW |

### 6.2 Operating Modes

| Mode | Components Active | Power Draw | Runtime (250 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising | 5-15 mW | 40-120 hours |
| **Idle** | BLE connected, sensors idle | 20-50 mW | 12-30 hours |
| **Standard** | Radar + IMU | 100-300 mW | 3-9 hours |
| **Active detection** | All + haptic bursts | 200-500 mW | 2-4.5 hours |

### 6.3 Haptic Power Consumption

| Haptic Type | Pulse Duration | Energy per Pulse | Pulses per Battery (250 mAh) |
|-------------|---------------|------------------|------------------------------|
| LRA (typical) | 100-300 ms | 30-100 mJ | 30,000-100,000 |
| Piezo | 1-10 ms | 1-10 mJ | 300,000-3,000,000 |

**Haptic power is negligible for overall battery budget.** Even thousands of haptic alerts consume minimal energy compared to continuous radar operation.

---

## 7. Anklet Node Power Architecture

### 7.1 Component Power Consumption

| Component | Idle | Active | Peak |
|-----------|------|--------|------|
| **ToF/LiDAR** | 20-50 mW | 100-200 mW | 300 mW |
| **IMU** | 1-5 mW | 10-50 mW | 80 mW (high-rate) |
| **Haptic** | 0 mW | 100-300 mW | 500 mW |
| **BLE Radio** | 5-15 mW | 30-80 mW | 120 mW |
| **Optional mmWave** | 30-80 mW | 100-250 mW | 400 mW |

### 7.2 Operating Modes

| Mode | Components Active | Power Draw | Runtime (400 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising | 5-15 mW | 60-180 hours |
| **Standard** | ToF + IMU | 50-150 mW | 10-30 hours |
| **Enhanced** | ToF + IMU + mmWave | 150-400 mW | 4-12 hours |

### 7.3 Gait Analysis Power Optimization

Gait analysis requires IMU at 200 Hz minimum. Power optimization strategies:

**On-node gait event detection:**
- Detect steps on-node using IMU data
- Transmit gait events (once per step) instead of raw IMU
- Raw IMU burst only on anomaly detection
- Reduces BLE bandwidth and processing power

**Power savings:**

| Mode | Data Transmitted | Power Draw |
|------|------------------|------------|
| Raw IMU stream | 200 Hz × 6 axes × 2 bytes = 2.4 KB/s | 50-80 mW |
| Gait events | ~2 events/s × 20 bytes = 40 B/s | 15-30 mW |
| **Savings** | **98% bandwidth reduction** | **60-70% power reduction** |

---

## 8. Eyewear Node Power Architecture

### 8.1 Component Power Consumption

| Component | Idle | Active | Peak |
|-----------|------|--------|------|
| **Event Camera** | 5-20 mW | 50-150 mW | 300 mW |
| **Conventional Camera** | 20-50 mW | 150-400 mW | 600 mW |
| **IMU** | 1-5 mW | 5-20 mW | 30 mW |
| **BLE Radio** | 5-15 mW | 30-80 mW | 120 mW |

### 8.2 Operating Modes

| Mode | Components Active | Power Draw | Runtime (80 mAh) |
|------|-------------------|------------|-------------------|
| **Sleep** | BLE advertising | 5-15 mW | 20-60 hours |
| **Event-only** | Event camera + IMU | 50-150 mW | 2-6 hours |
| **Event + camera** | All | 200-500 mW | 0.5-2 hours |

### 8.3 Extended Runtime Options

**Option A — Belt power cable:**
- Thin cable from belt to eyewear
- Unlimited runtime while connected
- Best for fixed-position or low-activity monitoring

**Option B — Larger battery:**
- 150 mAh battery (increases form factor)
- 1-3 hours runtime with event camera

**Option C — Duty cycling:**
- Event camera on only during high-risk situations
- User-triggered or context-triggered activation
- Extends runtime to 8-16 hours

---

## 9. Power Profiles

### 9.1 Definition

Power profiles are named configurations that balance capability vs. battery life:

| Profile | Description | Battery Impact |
|---------|-------------|----------------|
| **Ultra-Low-Latency** | Maximum performance, shortest latency | Highest power, shortest runtime |
| **Performance** | Full sensing, standard latency | High power, moderate runtime |
| **Balanced** | Standard operation | Moderate power, all-day runtime |
| **Power-Saver** | Reduced sensing, longer intervals | Low power, multi-day runtime |
| **Sleep** | Minimal operation | Lowest power, maximum runtime |

### 9.2 Profile Configuration

```toml
[power.profile]
active = "balanced"               # "ultra_low_latency" | "performance" | "balanced" | "power_saver" | "sleep"
auto_profile = true               # Automatically adjust based on context

[power.profile.ultra_low_latency]
# Maximum performance for critical situations
description = "All sensors active, lowest latency, maximum alert speed"
ble_connection_interval_ms = 7.5
lidar_update_hz = 50
slam_enabled = true
camera_resolution = "1080p"
camera_fps = 30
expected_runtime_hours = 2

[power.profile.performance]
# Full sensing capability, standard latency
description = "Full sensing capability, standard latency"
ble_connection_interval_ms = 15
lidar_update_hz = 20
slam_enabled = true
camera_resolution = "720p"
camera_fps = 15
expected_runtime_hours = 6

[power.profile.balanced]
# Default everyday operation
description = "Standard everyday operation, all-day runtime"
ble_connection_interval_ms = 30
lidar_update_hz = 10
slam_enabled = false               # Sparse tracking only
camera_resolution = "vga"
camera_fps = 10
expected_runtime_hours = 12

[power.profile.power_saver]
# Extended runtime, reduced sensing
description = "Extended runtime, reduced sensing"
ble_connection_interval_ms = 100
lidar_update_hz = 5
slam_enabled = false
camera_enabled = false
expected_runtime_hours = 36

[power.profile.sleep]
# Minimal operation
description = "Minimal operation, maximum runtime"
ble_connection_interval_ms = 1000
sensors_enabled = false
expected_runtime_hours = 72
```

### 9.3 Automatic Profile Switching

```toml
[power.auto_profile]
enabled = true

[power.auto_profile.triggers]
# Switch to ultra-low-latency when extreme velocity detected
extreme_velocity_detected = "ultra_low_latency"

# Switch to performance when threat detected
threat_detected = "performance"

# Switch to balanced when active
user_active = "balanced"

# Switch to power-saver when idle
idle_after_minutes = 15
idle_profile = "power_saver"

# Switch to sleep when stationary for extended period
sleep_after_minutes = 60
sleep_profile = "sleep"

# Battery-based profile switching
[power.auto_profile.battery]
# Gradually reduce capability as battery depletes
thresholds = [
    { percent = 50, profile = "balanced" },
    { percent = 25, profile = "power_saver" },
    { percent = 10, profile = "sleep" },
]
```

---

## 10. Battery Sizing and Runtime Estimation

### 10.1 Battery Capacity Reference

| Node | Battery Options | Capacity Range |
|------|-----------------|-----------------|
| Pendant (Standard) | 1S LiPo | 150-300 mAh |
| Pendant (360°) | 1S LiPo, distributed | 400-1000 mAh |
| Pendant (Medallion) | 1S LiPo | 800-1500 mAh |
| Bracelet | 1S LiPo | 150-500 mAh |
| Belt (MCU) | 1S LiPo | 2000 mAh |
| Belt (Linux SoM) | 1S LiPo, multi-cell | 5000-7000 mAh |
| Belt (Hot-swappable) | 2 × 1S LiPo | 5000-7000 mAh each |
| Anklet | 1S LiPo | 300-500 mAh |
| Eyewear | 1S LiPo | 50-150 mAh |

### 10.2 Runtime Calculation Formula

```
Runtime (hours) = (Battery_mAh × Voltage_V) / Power_W × 1000

Example: Linux SoM Belt in Standard Mode
Battery: 5000 mAh × 3.7V = 18.5 Wh
Power: 0.75 W
Runtime: 18.5 / 0.75 = 24.7 hours theoretical

Real-world derating: × 0.85 (efficiency, aging) = 21 hours
```

### 10.3 Runtime Tables by Node and Mode

#### Belt Node (Linux SoM, 5000 mAh)

| Mode | Power (W) | Theoretical Runtime | Real-World Runtime |
|------|-----------|---------------------|-------------------|
| Sleep | 0.05 | 370 hours | ~300 hours |
| Idle | 0.3 | 62 hours | ~50 hours |
| Standard | 0.75 | 25 hours | 20-22 hours |
| Active | 2.5 | 7.4 hours | 6-7 hours |
| Streaming | 4.0 | 4.6 hours | 3.5-4.5 hours |
| Maximum | 5.3 | 3.5 hours | 2.5-3 hours |

#### Pendant (360°, 800 mAh)

| Mode | Power (W) | Theoretical Runtime | Real-World Runtime |
|------|-----------|---------------------|-------------------|
| Sparse sensing | 0.3 | 9.9 hours | 8-9 hours |
| Cameras QVGA | 1.5 | 2.0 hours | 1.5-2 hours |
| Cameras VGA | 2.5 | 1.2 hours | 1 hour |
| Cameras 720p | 3.5 | 0.85 hours | 40-50 min |
| Full active | 4.5 | 0.66 hours | 30-40 min |

#### Bracelet (250 mAh)

| Mode | Power (W) | Theoretical Runtime | Real-World Runtime |
|------|-----------|---------------------|-------------------|
| Sleep | 0.01 | 92 hours | 75-85 hours |
| Standard | 0.2 | 4.6 hours | 4-5 hours |
| Active | 0.4 | 2.3 hours | 2-2.5 hours |

---

## 11. Thermal Management

### 11.1 Thermal Constraints

**Skin-contact temperature limits:**
- 35-38°C: Comfortable
- 38-40°C: Noticeable warmth, acceptable for extended wear
- 40-42°C: Maximum acceptable limit
- > 42°C: Uncomfortable, risk of low-temperature burns with prolonged contact

**Heat generation sources:**

| Component | Max Heat Output | Typical in Operation |
|-----------|-----------------|---------------------|
| Vision processor (pendant) | 2 W | 0.5-1.5 W |
| Linux SoM (belt) | 7 W | 1-4 W |
| mmWave radar | 0.6 W | 0.2-0.4 W |
| Camera array (8 cameras) | 2 W | 0.5-1.5 W |
| Cellular module | 1.2 W | 0.3-0.8 W |

### 11.2 Thermal Mitigation Strategies

#### Passive Cooling (All Nodes)

| Strategy | Implementation | Effectiveness |
|----------|----------------|---------------|
| Thermal vias | Vias under hot components to spread heat | Moderate |
| Air gap | Gap between PCB and enclosure | Moderate |
| Thermal pad | Pad between IC and enclosure | Good |
| Enclosure material | Metal for spreading, plastic for insulation | Depends on design |

#### Active Cooling (Belt Node Only)

| Strategy | Implementation | Effectiveness |
|----------|----------------|---------------|
| Vented enclosure | Vent holes on non-skin side | Good |
| Heat spreader | Aluminum plate under SoM | Good |
| Active fan | Small blower (research variant) | Excellent |

### 11.3 Thermal Monitoring

All nodes should monitor enclosure temperature:

```toml
[thermal]
monitoring_enabled = true
thermistor_pin = "NTC1"           # NTC thermistor on enclosure interior
sampling_interval_ms = 1000

[thermal.thresholds]
warning_c = 40                    # Begin throttling
throttle_c = 42                   # Reduce processing
critical_c = 45                   # Disable high-power functions
shutdown_c = 50                   # Safe shutdown

[thermal.throttling]
reduce_slam_rate_percent = 50     # Reduce SLAM frame rate
reduce_camera_fps_percent = 50    # Reduce camera frame rate
disable_features = ["streaming"]  # Disable high-power features
```

### 11.4 Thermal Response Time

| Node | Thermal Mass | Time to Equilibrium (2W load) | Time to 42°C (2W load) |
|------|--------------|------------------------------|------------------------|
| Pendant (Standard) | Low | 2-3 minutes | 5-8 minutes |
| Pendant (360°) | Medium | 3-5 minutes | 8-12 minutes |
| Bracelet | Very low | 1-2 minutes | 3-5 minutes |
| Belt (small) | Medium | 5-8 minutes | 15-25 minutes |
| Belt (large) | High | 8-15 minutes | 25-40 minutes |

### 11.5 Thermal Budgets by Activity

**Example: 360° Pendant active streaming**

| Time (min) | Power (W) | Enclosure Temp | Action |
|------------|-----------|----------------|--------|
| 0-2 | 3.0 | 25°C | Starting |
| 2-5 | 3.0 | 30°C | Warming |
| 5-10 | 3.0 | 35°C | Comfortable |
| 10-20 | 3.0 | 38°C | Warm |
| 20-30 | 2.5 (throttled) | 40°C | Warning, throttle begins |
| 30-45 | 2.0 (further throttled) | 41°C | Reduced performance |
| 45+ | 1.5 | 41°C | Stable at reduced power |

---

## 12. Charging Architecture

### 12.1 Charging Methods by Node

| Node | Charging Method | Typical Charge Time |
|------|-----------------|---------------------|
| Pendant (Standard) | USB-C, Qi wireless, magnetic pogo | 1-2 hours |
| Pendant (360°) | USB-C, magnetic pogo, belt power | 2-3 hours |
| Bracelet | Qi wireless, magnetic pogo | 1-1.5 hours |
| Belt (MCU) | USB-C PD | 2-3 hours |
| Belt (Linux SoM) | USB-C PD | 3-5 hours |
| Belt (Hot-swap) | USB-C PD, hot-swap | 2-3 hours per battery |
| Anklet | Qi wireless, magnetic pogo | 1-2 hours |
| Eyewear | USB-C, magnetic pogo | 0.5-1 hour |

### 12.2 USB-C Power Delivery

**Belt node (primary charging):**
- USB-C PD preferred (5-20 V, up to 3 A)
- Faster charging: 15W (5V/3A or 9V/1.67A)
- Supports charging while operating

**Smaller nodes:**
- USB-C basic (5V/1A)
- Qi wireless (5W typical)
- Magnetic pogo dock (5V/1A)

### 12.3 Charging Configuration

```toml
[charging]
method = "usb_pd"                 # "usb_basic" | "usb_pd" | "qi" | "pogo" | "belt_power"

[charging.usb_pd]
max_voltage_v = 9                  # 5, 9, 12, 15, or 20 V
max_current_a = 2.0

[charging.qi]
power_w = 5                        # 5 or 10 W

[charging.belt_power]
enabled = true                     # For pendant/eyewear receiving power from belt
voltage_v = 5.0
max_current_ma = 500

[charging.protection]
overcharge_cutoff_v = 4.25
overdischarge_cutoff_v = 3.0
temperature_cutoff_c = 45          # Stop charging if battery > 45°C
```

### 12.4 Hot-Swap Charging (Belt Variant E)

**Architecture:**
```
Battery Bay 1 ──┐
                ├── Power Path Controller ── System Load
Battery Bay 2 ──┘

Operation:
1. Battery 1 powers system
2. Battery 2 can be removed and replaced
3. Power path controller switches to Battery 2 if Battery 1 removed
4. No interruption to system operation
```

**Implementation:**
- Dedicated power path management IC (e.g., TI BQ25895, MPS MP2650)
- Automatic switchover when battery removed
- Simultaneous charging of both batteries when docked

**Configuration:**
```toml
[charging.hot_swap]
enabled = true
bay_count = 2
automatic_switchover = true
simultaneous_charging = true
```

---

## 13. Battery Safety

### 13.1 Protection Requirements

All LiPo batteries must have:

| Protection | Threshold | Implementation |
|------------|-----------|-----------------|
| Overcharge | 4.25 V | MOSFET cutoff |
| Overdischarge | 3.0 V | MOSFET cutoff |
| Overcurrent | Per cell spec | PTC fuse or IC |
| Short circuit | Immediate | IC protection |
| Thermal | 60°C | Thermistor + IC |

### 13.2 Protection Circuit

```
Battery Cell
     │
     ├── Protection IC (DW01 or equivalent)
     │       ├── Overcharge detection
     │       ├── Overdischarge detection
     │       └── Short circuit detection
     │
     └── MOSFET (dual N-channel)
             └── Cuts charge/discharge path on fault
```

### 13.3 Storage and Handling

**Long-term storage (> 1 month):**
- Charge to 40-60% capacity
- Store at 15-25°C
- Recharge every 3-6 months

**Handling:**
- Never puncture or crush
- Never expose to fire
- Never charge below 0°C
- Stop use if swelling or damage observed

---

## 14. Power Budget Optimization Strategies

### 14.1 Strategy 1: Sensing Duty Cycling

**Principle:** Sensors don't need to be 100% active to be useful.

| Sensor | Duty Cycle Option | Power Savings | Impact on Detection |
|--------|-------------------|----------------|---------------------|
| LiDAR | 10 Hz instead of 50 Hz | 80% | Misses fast transients |
| mmWave | 50% duty cycle | 50% | Delay in detection |
| Camera | 5 fps instead of 30 fps | 83% | Reduced temporal resolution |

**Configuration:**
```toml
[power.duty_cycling]
enabled = true

[power.duty_cycling.sensors]
lidar_hz = 10                      # Reduce from 50 Hz
mmwave_duty_percent = 75           # Active 75% of time
camera_fps = 15                    # Reduce from 30
```

### 14.2 Strategy 2: Event-Triggered Activation

**Principle:** Activate high-power sensors only when needed.

| Sensor | Trigger | Power Savings |
|--------|---------|----------------|
| LiDAR | Event camera fast motion detected | ~70% (LiDAR off until needed) |
| Conventional camera | Presence detected by radar | ~90% (camera off until presence) |
| SLAM | User in unknown area | ~50% (SLAM off in known areas) |

### 14.3 Strategy 3: Adaptive Streaming

**Principle:** Adjust streaming quality based on network and battery.

| Condition | Streaming Quality | Power Impact |
|-----------|-------------------|--------------|
| Battery > 50% | High (1080p) | Baseline |
| Battery 25-50% | Medium (720p) | ~30% reduction |
| Battery 10-25% | Low (480p) | ~60% reduction |
| Battery < 10% | Metadata only | ~95% reduction |

### 14.4 Strategy 4: Network Selection

**Principle:** Use lowest-power network that meets needs.

| Network | Power | Use Case |
|---------|-------|----------|
| BLE | 5-20 mW | Metadata, alerts, low-bandwidth |
| WiFi | 100-500 mW | High-bandwidth streaming |
| Cellular | 150-1200 mW | Remote access only |

**Strategy:** Prefer BLE for control and metadata. Use WiFi only for streaming. Use cellular only when remote access is required.

---

## 15. Battery State Estimation

### 15.1 Fuel Gauge

All nodes should have accurate battery state-of-charge (SoC) estimation:

| Method | Accuracy | Cost |
|--------|----------|------|
| Voltage-based | ±15% | Minimal |
| Coulomb counting | ±5% | Moderate (requires current sense) |
| Impedance tracking | ±3% | Higher (requires dedicated IC) |

**Recommended:** Dedicated fuel gauge IC (e.g., TI BQ27441, Maxim MAX17048)

### 15.2 SoC Configuration

```toml
[battery.fuel_gauge]
method = "coulomb_counting"        # "voltage" | "coulomb_counting" | "impedance"
reporting_interval_s = 60
low_battery_percent = 20           # Warning threshold
critical_battery_percent = 10      # Critical threshold
shutdown_percent = 5               # Graceful shutdown
```

### 15.3 Battery Health Monitoring

```toml
[battery.health]
cycle_count_enabled = true
capacity_tracking = true
aging_model = "simple"            # "simple" | "advanced"

[battery.health.alerts]
capacity_degradation_percent = 20  # Alert when capacity < 80% of original
high_cycle_count = 300            # Alert at 300 cycles
```

---

## 16. Power-Performance Tradeoffs

### 16.1 Capability vs. Runtime Matrix

| Capability Level | Belt Runtime (5000 mAh) | Pendant 360° Runtime (800 mAh) |
|------------------|------------------------|--------------------------------|
| **Maximum** (all features) | 2-4 hours | 30-60 minutes |
| **High** (SLAM, streaming) | 4-8 hours | 1-2 hours |
| **Standard** (sparse tracking) | 12-20 hours | 4-8 hours |
| **Extended** (reduced sensing) | 24-36 hours | 10-20 hours |
| **Maximum** (minimal sensing) | 48-72 hours | 24-48 hours |

### 16.2 Feature Power Impact

| Feature | Additional Power | Runtime Impact |
|---------|-------------------|----------------|
| Dense SLAM | +2 W | -50% runtime |
| 360° streaming | +3 W | -65% runtime |
| Cellular streaming | +1 W | -20% runtime |
| Extreme velocity detection | +0.5 W | -10% runtime |
| 720p vs VGA camera | +0.5 W | -20% runtime |
| 30 fps vs 10 fps | +0.3 W | -15% runtime |

---

## 17. Emergency Power Behavior

### 17.1 Low Battery Actions

```toml
[battery.emergency]
level_20_percent = "warning"      # Notify user
level_15_percent = "reduce_features"
level_10_percent = "critical_warning"
level_5_percent = "shutdown_preparation"
level_3_percent = "graceful_shutdown"

[battery.emergency.feature_reduction]
# Features to disable at each battery level
level_15_percent = ["slam", "streaming"]
level_10_percent = ["camera", "uwb"]
level_5_percent = ["radar_high_rate", "haptic_alerts"]
```

### 17.2 Graceful Shutdown

When battery reaches critical level:
1. Log current state
2. Save SLAM map to SD card
3. Transmit final status to belt (if possible)
4. Disable all sensors
5. Enter deep sleep
6. Wake only on charger connection

---

## 18. Configuration Reference

### 18.1 Complete Power Configuration

```toml
# sentinel-wear.toml

[power]
profile = "balanced"
auto_profile = true

[power.profile]
active = "balanced"
auto_profile = true

[power.profile.ultra_low_latency]
ble_connection_interval_ms = 7.5
lidar_update_hz = 50
slam_enabled = true
camera_resolution = "1080p"
camera_fps = 30
expected_runtime_hours = 2

[power.profile.performance]
ble_connection_interval_ms = 15
lidar_update_hz = 20
slam_enabled = true
camera_resolution = "720p"
camera_fps = 15
expected_runtime_hours = 6

[power.profile.balanced]
ble_connection_interval_ms = 30
lidar_update_hz = 10
slam_enabled = false
camera_resolution = "vga"
camera_fps = 10
expected_runtime_hours = 12

[power.profile.power_saver]
ble_connection_interval_ms = 100
lidar_update_hz = 5
slam_enabled = false
camera_enabled = false
expected_runtime_hours = 36

[power.profile.sleep]
ble_connection_interval_ms = 1000
sensors_enabled = false
expected_runtime_hours = 72

[power.auto_profile]
enabled = true

[power.auto_profile.triggers]
extreme_velocity_detected = "ultra_low_latency"
threat_detected = "performance"
user_active = "balanced"
idle_after_minutes = 15
idle_profile = "power_saver"
sleep_after_minutes = 60
sleep_profile = "sleep"

[power.auto_profile.battery]
thresholds = [
    { percent = 50, profile = "balanced" },
    { percent = 25, profile = "power_saver" },
    { percent = 10, profile = "sleep" },
]

[power.duty_cycling]
enabled = true

[power.duty_cycling.sensors]
lidar_hz = 10
mmwave_duty_percent = 75
camera_fps = 15

[thermal]
monitoring_enabled = true
thermistor_pin = "NTC1"
sampling_interval_ms = 1000

[thermal.thresholds]
warning_c = 40
throttle_c = 42
critical_c = 45
shutdown_c = 50

[thermal.throttling]
reduce_slam_rate_percent = 50
reduce_camera_fps_percent = 50
disable_features = ["streaming"]

[charging]
method = "usb_pd"

[charging.usb_pd]
max_voltage_v = 9
max_current_a = 2.0

[charging.protection]
overcharge_cutoff_v = 4.25
overdischarge_cutoff_v = 3.0
temperature_cutoff_c = 45

[battery.fuel_gauge]
method = "coulomb_counting"
reporting_interval_s = 60
low_battery_percent = 20
critical_battery_percent = 10
shutdown_percent = 5

[battery.health]
cycle_count_enabled = true
capacity_tracking = true
aging_model = "simple"

[battery.emergency]
level_20_percent = "warning"
level_15_percent = "reduce_features"
level_10_percent = "critical_warning"
level_5_percent = "shutdown_preparation"
level_3_percent = "graceful_shutdown"

[battery.emergency.feature_reduction]
level_15_percent = ["slam", "streaming"]
level_10_percent = ["camera", "uwb"]
level_5_percent = ["radar_high_rate", "haptic_alerts"]
```

---

## 19. Summary

**Key principles:**
1. **Power is the primary constraint** — every feature decision has a runtime cost
2. **Thermal comfort is non-negotiable** — skin temperature must stay below 42°C
3. **Graceful degradation** — the system should remain useful as battery depletes
4. **User-configurable tradeoffs** — users choose capability vs. runtime
5. **Extended runtime options** — belt power input and hot-swap batteries for continuous operation

**Runtime expectations:**
- Standard belt node (Linux SoM): 10-20 hours (sparse), 3-6 hours (active)
- 360° pendant: 2-6 hours (active cameras), extended indefinitely with belt power
- Bracelets: 20-40 hours (standard), 2-4 hours (with camera)
- Anklets: 30-60 hours (standard), 10-30 hours (extended)

**Power optimization hierarchy:**
1. Maximize detection range (more important than latency for realistic scenarios)
2. Use CW radar for Tier 1 (lowest power for fast detection)
3. Duty cycle high-power sensors
4. Event-triggered activation
5. Adaptive streaming quality
6. Network selection (BLE preferred, WiFi only for streaming)

---

**End of Document**
