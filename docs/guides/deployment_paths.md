# Deployment Paths — SENTINEL-WEAR

**Purpose:** Complete reference guide for selecting and configuring SENTINEL-WEAR variants based on use case, environment, and capability requirements.

**Audience:** Researchers, integrators, security professionals, accessibility users, and developers.

---

## Table of Contents

1. [Understanding Hardware Variants vs. Configuration Options](#1-understanding-hardware-variants-vs-configuration-options)
2. [Node Types and Variants Summary](#2-node-types-and-variants-summary)
3. [Deployment Path Overview](#3-deployment-path-overview)
4. [Path 1: Basic Awareness](#4-path-1-basic-awareness)
5. [Path 2: Standard Personal Awareness](#5-path-2-standard-personal-awareness)
6. [Path 3: Security Professional](#6-path-3-security-professional)
7. [Path 4: Extended Runtime / Continuous Use](#7-path-4-extended-runtime--continuous-use)
8. [Path 5: Extreme Velocity Detection](#8-path-5-extreme-velocity-detection)
9. [Path 6: Accessibility (Visually Impaired)](#9-path-6-accessibility-visually-impaired)
10. [Path 7: Research Platform](#10-path-7-research-platform)
11. [Path 8: Long-Range Detection](#11-path-8-long-range-detection)
12. [Path 9: Close-Range Urban](#12-path-9-close-range-urban)
13. [Path 10: Dual-Mode (Awareness + Detection)](#13-path-10-dual-mode-awareness--detection)
14. [Decision Tree for Path Selection](#14-decision-tree-for-path-selection)
15. [Configuration Reference by Path](#15-configuration-reference-by-path)
16. [Cost and Complexity Comparison](#16-cost-and-complexity-comparison)
17. [Runtime Estimates by Path](#17-runtime-estimates-by-path)
18. [Connectivity Architecture for All Paths](#18-connectivity-architecture-for-all-paths)
19. [Upgrade Paths](#19-upgrade-paths)
20. [Legal and Compliance Considerations](#20-legal-and-compliance-considerations)

---

## 1. Understanding Hardware Variants vs. Configuration Options

### The Critical Distinction

When building a SENTINEL-WEAR system, understanding the difference between **hardware variants** and **configuration options** is essential for cost, procurement, and flexibility.

**Hardware Variant:** A physically different PCB design or major component change that must be selected at the time of manufacturing or purchase. Different variants have different capabilities that cannot be changed after manufacture.

**Configuration Option:** A setting or population choice on the same PCB design. Can be changed at assembly time or, in some cases, after deployment through software configuration or modular component addition.

### Hardware Variants Require Different PCBs

| Variant Type | What Changes | When Decided |
|--------------|--------------|--------------|
| Belt MCU vs Linux SoM | Different compute platform | At manufacture |
| Pendant Standard vs 360° Curved | Different PCB architecture (flat vs flex) | At manufacture |
| Pendant Standard vs Medallion | Different form factor | At manufacture |
| Eyewear Clip-on vs Frame-integrated | Different PCB shape | At manufacture |

### Configuration Options Work on Same PCB

| Configuration Option | What Changes | When Decided |
|---------------------|--------------|--------------|
| Camera present/absent | Module population | At assembly or later |
| UWB enabled | Module population or firmware config | At assembly or later |
| Battery size | Cell selection | At assembly |
| Microphone count | Component population | At assembly |
| Piezo vs LRA haptic | Component selection | At assembly |
| Storage target | Software config | Anytime |
| Recording mode | Software config | Anytime |
| Detection range optimization | Software/firmware config | Anytime |

### Why This Matters

**Implication for deployment paths:**
- A "Path 3: Security Professional" could be built from:
  - Belt node PCB (Linux SoM variant)
  - Pendant PCB (360° Curved variant)
  - These are hardware variant choices
- But the camera configuration, storage settings, and detection tuning are configuration options
- Same hardware can serve different paths with different configurations

---

## 2. Node Types and Variants Summary

### 2.1 Pendant Node

| Hardware Variant | Description | Key Differentiators |
|------------------|-------------|---------------------|
| **A — Standard Flat** | Compact medallion form | Flat PCB, lowest cost, basic sensing |
| **B — 360° Curved** | Curved flexible PCB with camera array | 4-8 cameras, 360° capture, wired internal bus |
| **C — Medallion** | Larger premium form | Higher compute, more sensors, larger battery |
| **D — Tactical/Extended Runtime** | Larger with belt power input | Up to 2000 mAh, external power |
| **E — Event-Enhanced** | Standard or 360° + event cameras | Event cameras for fast transient detection |
| **F — Long-Range Detection** | Optimized for maximum detection range | Higher-power radar, directional antenna |

**Configuration Options (Apply to All Variants):**

| Option | Values | Notes |
|--------|--------|-------|
| Camera | None / Single / Wide-angle | PCB has footprint, module optionally populated |
| UWB | Disabled / Enabled | For bandwidth augmentation and precision timing |
| Battery | 150 / 250 / 400 mAh (Standard) or 400-1000 mAh (360°) | Different cell sizes |
| Microphones | 3 / 4 / 6 element | Same footprint, different population |
| Extreme Velocity Detection | Disabled / Standard / Long-Range | CW radar configuration |

### 2.2 Bracelet Node

**Single Hardware Design** supports all configurations via component population:

| Configuration | Sensors | Camera | UWB | Battery |
|---------------|---------|--------|-----|---------|
| Minimal | mmWave + IMU + Haptic | No | No | 150 mAh |
| Standard | mmWave + IMU + Haptic + ToF (optional) | No | Optional | 250 mAh |
| Extended | All sensors + camera | Forward-facing | Optional | 400 mAh |

**Configuration Options:**

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Standard / Extended | Different population |
| Camera | None / Forward-facing / Wide-angle | Optional module |
| UWB | Disabled / Enabled | Optional module |
| ToF | Absent / Present | Optional short-range sensing |

### 2.3 Belt Node

| Hardware Variant | Compute | Storage | Battery | Runtime (Sparse) |
|------------------|---------|---------|---------|------------------|
| **A — MCU-Class** | STM32H7 (Cortex-M7, 480 MHz) | SD card | 2000 mAh | 15-20 hours |
| **B — Linux SoM** | Raspberry Pi CM4 / NXP i.MX 8M Plus | SD/NVMe | 5000-7000 mAh | 10-15 hours |
| **C — High-Performance** | Qualcomm SA8155P / NPU class | NVMe | 5000-7000 mAh | 8-12 hours |
| **D — ESP32-Based** | ESP32-S3 | SD card | 2000 mAh | 12-16 hours |
| **E — Hot-Swappable** | Linux SoM | SD/NVMe | 5000-7000 mAh × 2 | Unlimited (with swaps) |

**Configuration Options (All Variants):**

| Option | Values | Notes |
|--------|--------|-------|
| Cellular module | None / LTE Cat 1 / LTE Cat 4 / 5G | Based on remote access needs |
| UWB | Disabled / Enabled | For precision timing with nodes |
| WiFi band | 5 GHz preferred / 2.4 GHz only | For RF coexistence |
| Antenna diversity | Single / Dual | Dual antennas improve reliability |
| Haptic type | LRA / Piezo / Both | Piezo for faster onset |
| Piezo count | 0 / 2 / 4 | Directional haptic array |

### 2.4 Anklet Node

**Single Hardware Design** supports all configurations:

| Configuration | Sensors | Camera | Battery |
|---------------|---------|--------|---------|
| Minimal | ToF + IMU + Haptic | No | 300 mAh |
| Extended | ToF + LiDAR + mmWave + IMU + Haptic | Downward (optional) | 500 mAh |

**Configuration Options:**

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Extended | Different population |
| Camera | None / Downward-facing | Ground obstacle detection |
| UWB | Disabled / Enabled | Recommended for gait sync |

### 2.5 Eyewear Node

| Hardware Variant | Form Factor | Sensors | Battery | Notes |
|------------------|-------------|---------|---------|-------|
| **A — Event-Only Clip-on** | Clips to existing glasses | Event camera + IMU | 50-80 mAh | Fast transient detection only |
| **B — Event + Camera Clip-on** | Clips to existing glasses | Event + conventional + IMU | 50-80 mAh | Visual capture + fast detection |
| **C — Frame-Integrated** | Embedded in temple arms | Event + cameras + IMU | 80-150 mAh | Requires custom frames |
| **D — Headband** | Central forehead PCB | Full sensor array | 80-150 mAh | Easier prototype |

**Configuration Options:**

| Option | Values | Notes |
|--------|--------|-------|
| Sensor type | Event-only / Event + Camera | Event-only for fast detection |
| Camera count | 1 / 2 / 3 | Forward + side cameras |
| UWB | Disabled / Enabled | Head-torso precision sync |
| Power source | Battery / Belt cable | Cable for extended runtime |

---

## 3. Deployment Path Overview

### Quick Reference Matrix

| Path | Primary Use Case | Belt Variant | Pendant Variant | Key Features | Runtime |
|------|------------------|--------------|-----------------|--------------|----------|
| 1 — Basic Awareness | Minimal cost, basic presence | A or D | A | Sparse tracking only | 12-20 hrs |
| 2 — Standard Personal | Consumer full capability | B | A + camera | Sparse + SLAM, streaming | 10-15 hrs |
| 3 — Security Professional | 360° + evidence | B or C | B (360°) | Full capability | 5-8 hrs active |
| 4 — Extended Runtime | 24/7 operation | E | D | Hot-swap batteries | Unlimited |
| 5 — Extreme Velocity | Fast threat detection | B | E | Event + CW radar | 8-12 hrs |
| 6 — Accessibility | Visually impaired | B | B (360°) | Haptic spatial awareness | 10-15 hrs |
| 7 — Research | Experimentation | Any | Any | Maximum flexibility | Variable |
| 8 — Long-Range | Rifle detection at 100m+ | B | F | Max detection range | 8-12 hrs |
| 9 — Close-Range Urban | Handgun detection 5-15m | A | E | Fast local alert | 15-20 hrs |
| 10 — Dual-Mode | Everyday + safety | B | B (360°) + CW radar | Full awareness + detection | 5-8 hrs active |

---

## 4. Path 1: Basic Awareness

### 4.1 Target User

- Casual user wanting basic presence awareness
- Minimal investment
- Long battery life priority
- No streaming or recording requirements

### 4.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant A (MCU-class) or D (ESP32) | Lowest cost; no SLAM needed |
| **Pendant** | Variant A, no camera, no UWB | Basic sensing |
| **Bracelets** | Minimal config (no camera, no UWB) | Basic coverage |
| **Anklets** | Minimal config | Basic gait detection |
| **Eyewear** | Not included | Not needed for this path |

### 4.3 Capabilities

| Capability | Status |
|------------|--------|
| Sparse tracking (PentaTrack) | ✅ Active |
| Dense SLAM | ❌ Not available (MCU belt) |
| Camera streaming | ❌ No cameras |
| 360° visual capture | ❌ Not available |
| Recording | ❌ Metadata only |
| Remote access | ❌ No cellular (optional WiFi only) |
| Extreme velocity detection | ❌ Not configured |

### 4.4 Expected Performance

| Metric | Value |
|--------|-------|
| Runtime (sparse mode) | 12-20 hours |
| Alert latency (standard BLE) | 20-60 ms |
| Tracking accuracy | Good (sparse model) |
| Detection types | Human, vehicle, animal presence |

### 4.5 Configuration File

```toml
# Path 1: Basic Awareness

[nodes]
belt = { variant = "mcu" }
pendant = { variant = "standard", camera = false, uwb = false }
bracelet_left = { config = "minimal" }
bracelet_right = { config = "minimal" }
anklet_left = { config = "minimal" }
anklet_right = { config = "minimal" }
eyewear = { enabled = false }

[policy]
default_mode = "PrivacyFirst"
tracking_mode = "sparse"

[data]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
recording_trigger = "none"

[extreme_velocity]
enabled = false
```

### 4.6 Estimated Cost

| Component | Relative Cost |
|-----------|---------------|
| Belt (MCU) | $ |
| Pendant (basic) | $ |
| Bracelets × 2 | $ each |
| Anklets × 2 | $ each |
| **Total** | $$ (Lowest) |

---

## 5. Path 2: Standard Personal Awareness

### 5.1 Target User

- Consumer wanting full awareness with streaming capability
- Balance of capability and battery life
- Occasional recording and evidence capture
- Home security enhancement

### 5.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B (Linux SoM) | SLAM, streaming, recording |
| **Pendant** | Variant A with camera | Forward visual capture |
| **Bracelets** | Standard config | Enhanced coverage |
| **Anklets** | Extended config | Enhanced gait analysis |
| **Eyewear** | Optional, Variant A (event-only) | Fast transient detection |

### 5.3 Capabilities

| Capability | Status |
|------------|--------|
| Sparse tracking (PentaTrack) | ✅ Active |
| Dense SLAM | ✅ Available (Linux SoM) |
| Camera streaming | ✅ Pendant camera |
| 360° visual capture | ❌ Not with standard pendant |
| Recording | ✅ On-demand |
| Remote access | ✅ WiFi to companion app |
| Extreme velocity detection | ⚠️ Optional (if configured) |

### 5.4 Expected Performance

| Metric | Sparse Mode | Full Active |
|--------|-------------|-------------|
| Runtime | 10-15 hours | 5-8 hours |
| Alert latency | 15-30 ms | 15-30 ms |
| Tracking accuracy | Good | Excellent (SLAM) |

### 5.5 Configuration File

```toml
# Path 2: Standard Personal Awareness

[nodes]
belt = { variant = "linux_som", battery_mah = 5000 }
pendant = { variant = "standard", camera = "forward" }
bracelet_left = { config = "standard", uwb = false }
bracelet_right = { config = "standard", uwb = false }
anklet_left = { config = "extended" }
anklet_right = { config = "extended" }
eyewear = { variant = "clip_on_event", enabled = false }

[policy]
default_mode = "PrivacyFirst"
tracking_mode = "sparse"            # Switch to "dense" for SLAM

[data]
store_raw_video = false
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "sd_card"
retention_days = 30
recording_trigger = "on_detection"

[extreme_velocity]
enabled = false
```

### 5.6 Estimated Cost

| Component | Relative Cost |
|-----------|---------------|
| Belt (Linux SoM) | $$$ |
| Pendant + camera | $$ |
| Bracelets × 2 (standard) | $$ total |
| Anklets × 2 (extended) | $$ total |
| **Total** | $$$$ (Moderate) |

---

## 6. Path 3: Security Professional

### 6.1 Target User

- Security personnel
- Private investigators
- Journalists in sensitive environments
- Professional users requiring 360° awareness and evidence capture

### 6.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B or C (Linux SoM with NPU option) | Full compute capability |
| **Pendant** | Variant B (360° Curved) with 8 cameras | Complete 360° visual capture |
| **Bracelets** | Extended with camera | Enhanced coverage + visual |
| **Anklets** | Extended config | Full gait analysis |
| **Eyewear** | Variant B or C | Forward visual context |

### 6.3 Capabilities

| Capability | Status |
|------------|--------|
| Sparse tracking (PentaTrack) | ✅ Active |
| Dense SLAM | ✅ Full capability |
| Camera streaming | ✅ Multiple cameras |
| 360° visual capture | ✅ 8-camera array |
| Recording | ✅ Continuous or triggered |
| Remote access | ✅ WiFi + optional cellular |
| Extreme velocity detection | ✅ Available |

### 6.4 Expected Performance

| Metric | Sparse Mode | Full Active |
|--------|-------------|-------------|
| Runtime | 10-12 hours | 5-8 hours |
| Alert latency | 10-20 ms (optimized BLE) | 10-20 ms |
| 360° streaming latency | N/A | 50-100 ms |
| SLAM quality | N/A | Excellent |

### 6.5 Configuration File

```toml
# Path 3: Security Professional

[nodes]
belt = { variant = "linux_som", battery_mah = 7000, piezo_haptic = true }
pendant_360 = { 
    variant = "curved", 
    camera_count = 8, 
    camera_resolution = "720p",
    event_camera_count = 2
}
bracelet_left = { config = "extended", camera = true }
bracelet_right = { config = "extended", camera = true }
anklet_left = { config = "extended", uwb = true }
anklet_right = { config = "extended", uwb = true }
eyewear = { variant = "frame_integrated", camera_count = 2 }

[policy]
default_mode = "SecurityFirst"
tracking_mode = "dense"

[data]
store_raw_video = true
store_raw_audio = true
store_raw_sensor_data = true
storage_target = "sd_card"
retention_days = 90
recording_trigger = "on_detection"
include_integrity_chain = true

[connectivity.cellular]
enabled = true
sim_type = "nano_sim"
stream_via_cellular = true

[extreme_velocity]
enabled = true
mode = "production"
detection_range_mode = "standard"

[haptic]
type = "both"
piezo_enabled = true
directional = true
actuator_count = 4
```

### 6.6 Estimated Cost

| Component | Relative Cost |
|-----------|---------------|
| Belt (Linux SoM + piezo) | $$$$ |
| Pendant 360° (8 cameras) | $$$$$ |
| Bracelets × 2 (extended + camera) | $$$ total |
| Anklets × 2 (extended + UWB) | $$$ total |
| Eyewear (frame-integrated) | $$$$ |
| **Total** | $$$$$$ (Highest standard path) |

---

## 7. Path 4: Extended Runtime / Continuous Use

### 7.1 Target User

- Users requiring 24/7 operation
- Security shifts
- Continuous monitoring deployments
- Field operations without charging access

### 7.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant E (Hot-swappable) | Dual battery bays |
| **Pendant** | Variant D (Tactical) or B (360°) | Extended battery or belt power input |
| **Bracelets** | Standard config | |
| **Anklets** | Standard config | |
| **Eyewear** | Optional | |

### 7.3 Key Differentiator: Hot-Swap Battery System

**Belt Variant E includes:**
- Two battery bays (each 5000-7000 mAh)
- Power path management IC
- Swap one battery while system runs on the other
- Unlimited runtime with periodic swaps

**Pendant power options:**
- Large internal battery (Variant D: up to 2000 mAh)
- Belt power input via conductive chain or cable
- Combination for maximum flexibility

### 7.4 Capabilities

| Capability | Status |
|------------|--------|
| Sparse tracking (PentaTrack) | ✅ Active |
| Dense SLAM | ✅ Available |
| 360° visual capture | ✅ With Variant B pendant |
| Recording | ✅ Continuous |
| Runtime | ✅ Unlimited (with battery swaps) |

### 7.5 Configuration File

```toml
# Path 4: Extended Runtime / Continuous Use

[nodes]
belt = { variant = "hot_swappable", battery_count = 2, battery_mah_each = 7000 }
pendant = { variant = "tactical", belt_power_input = true }
# Or: pendant_360 = { variant = "curved", belt_power_input = true }
bracelet_left = { config = "standard" }
bracelet_right = { config = "standard" }
anklet_left = { config = "standard" }
anklet_right = { config = "standard" }

[power]
hot_swap_enabled = true
low_battery_warning_percent = 30
critical_battery_percent = 15
auto_throttle_on_low_battery = true

[data]
continuous_recording = true
recording_trigger = "always"
```

### 7.6 Runtime Analysis

| Mode | Runtime Per Battery | With Hot-Swap |
|------|---------------------|----------------|
| Sparse tracking | 18-20 hours | Unlimited |
| Full active (SLAM + streaming) | 5-7 hours | Unlimited |

**Practical consideration:** User needs access to charged spare batteries and 2-3 hours to charge each.

---

## 8. Path 5: Extreme Velocity Detection

### 8.1 Target User

- Users in environments with potential for high-speed threats
- Security in active threat zones
- Personal protection enhancement (awareness layer)

### 8.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B (Linux SoM) + piezo haptics | Fast alert capability |
| **Pendant** | Variant E (Event-Enhanced) | Event cameras + CW radar |
| **Bracelets** | Standard + directional haptics | Directional alerts |
| **Anklets** | Standard | |
| **Eyewear** | Variant A (event-only) | Forward transient confirmation |

### 8.3 Understanding Extreme Velocity Detection

**Realistic engagement scenarios:**

| Weapon Type | Distance | Flight Time | Detection + Alert | Warning Time |
|-------------|----------|--------------|-------------------|--------------|
| Rifle (850 m/s) | 100 m | 117 ms | 0.65-2.4 ms | **115+ ms** |
| Rifle (850 m/s) | 50 m | 58 ms | 0.65-2.4 ms | **56+ ms** |
| Handgun (350 m/s) | 10 m | 28 ms | 0.65-2.4 ms | **26+ ms** |
| Handgun (350 m/s) | 5 m | 14 ms | 0.65-2.4 ms | **12+ ms** |

**Key insight:** At realistic rifle distances (50-100 m), warning times of 56-115 ms are achievable — well within human reaction time.

### 8.4 Tiered Detection Architecture

| Tier | Latency | Function | Output |
|------|---------|----------|--------|
| **Tier 1** | 50-150 µs | Reflex trigger (CW radar) | Binary detection, local haptic |
| **Tier 2** | 100-500 µs | Direction validation (event camera) | Direction vector |
| **Tier 3** | 5-50 ms | Characterization (FMCW, recording) | Full evidence |

### 8.5 Haptic Requirements for Extreme Velocity

| Actuator Type | Onset Time | Suitability |
|---------------|------------|-------------|
| LRA | 2-10 ms | ✅ Sufficient for rifle at 50+ m |
| Piezo | 0.5-1 ms | ✅ Optimal; additional margin |

**Recommendation:** Piezo for maximum margin, LRA acceptable for realistic rifle scenarios.

### 8.6 Configuration File

```toml
# Path 5: Extreme Velocity Detection

[nodes]
belt = { 
    variant = "linux_som", 
    piezo_haptic = true,
    antenna_diversity = true
}
pendant = { 
    variant = "event_enhanced",
    event_camera_count = 2,
    cw_radar = true
}
bracelet_left = { config = "standard", directional_haptic = true }
bracelet_right = { config = "standard", directional_haptic = true }
anklet_left = { config = "standard" }
anklet_right = { config = "standard" }
eyewear = { variant = "clip_on_event", enabled = true }

[extreme_velocity]
enabled = true
mode = "production"

[extreme_velocity.detection_range]
optimization_target = "balanced"
target_range_m = 100
latency_budget_ms = 2

[extreme_velocity.tier1]
velocity_threshold_ms = 50
local_haptic_trigger = true
haptic_type = "piezo"

[extreme_velocity.tier2]
enabled = true
direction_accuracy_deg = 15

[extreme_velocity.ble]
qos_class = "critical"
reserved_slot = true

[haptic.directional]
enabled = true
actuator_count = 4
alert_mode = "nearest_to_threat"
```

### 8.7 Expected Performance

| Scenario | Detection Time | Alert Time | Total Warning |
|----------|----------------|------------|---------------|
| Rifle at 100 m | 50-150 µs | 0.5-2 ms | 115+ ms |
| Rifle at 50 m | 50-150 µs | 0.5-2 ms | 56+ ms |
| Handgun at 10 m | 50-150 µs | 0.5-2 ms | 26+ ms |

---

## 9. Path 6: Accessibility (Visually Impaired)

### 9.1 Target User

- Visually impaired users
- Users with limited vision requiring environmental awareness
- Mobility assistance enhancement

### 9.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B (Linux SoM) | Rich audio processing for spatial description |
| **Pendant** | Variant B (360° Curved) | Full environmental context |
| **Bracelets** | Standard + directional haptics | Essential for direction indication |
| **Anklets** | Standard | Ground-level awareness |
| **Eyewear** | Not typically used | |

### 9.3 Accessibility-Specific Features

**Haptic Spatial Awareness:**
- Directional haptic alerts indicate obstacle/warning direction
- Left bracelet buzz = obstacle on left
- Right bracelet buzz = obstacle on right
- Both bracelets = obstacle ahead
- Anklets = ground-level obstacles

**Audio Scene Description:**
- Belt processes 360° visual data
- Generates audio descriptions of environment
- Obstacle proximity alerts via earpiece or bone conduction
- Spatial audio for direction cues

**Ground-Level Awareness:**
- Anklet ToF/LiDAR detects obstacles at floor level
- Stairs, curbs, trip hazards identified
- Haptic alert on approach

### 9.4 Capabilities

| Capability | Status |
|------------|--------|
| Sparse tracking (PentaTrack) | ✅ Active |
| Dense SLAM | ✅ Full 3D environment model |
| 360° visual capture | ✅ Continuous |
| Audio scene description | ✅ Primary output |
| Directional haptics | ✅ Primary navigation aid |
| Ground obstacle detection | ✅ Anklet sensing |

### 9.5 Configuration File

```toml
# Path 6: Accessibility (Visually Impaired)

[nodes]
belt = { variant = "linux_som", audio_processing = "enhanced" }
pendant_360 = { 
    variant = "curved", 
    camera_count = 8,
    camera_resolution = "720p"
}
bracelet_left = { config = "standard", directional_haptic = true }
bracelet_right = { config = "standard", directional_haptic = true }
anklet_left = { config = "extended" }
anklet_right = { config = "extended" }
eyewear = { enabled = false }

[accessibility]
audio_scene_description = true
spatial_audio = true
bone_conduction_output = true

[haptic.directional]
enabled = true
actuator_count = 4
intensity_level = "high"
patterns = ["navigation", "obstacle", "warning"]

[accessibility.ground_obstacles]
enabled = true
detection_range_m = 3.0
alert_method = "haptic_audio"
```

---

## 10. Path 7: Research Platform

### 10.1 Target User

- Academic researchers
- Industrial R&D teams
- Sensing algorithm developers
- Wearable computing researchers

### 10.2 Recommended Configuration

**Maximum flexibility configuration:**

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B or C (Linux SoM) | Maximum compute flexibility |
| **Pendant** | Any variant | Test different sensing configurations |
| **Bracelets** | Any variant | |
| **Anklets** | Any variant | |
| **Eyewear** | Any variant | |

### 10.3 Research-Specific Features

- All sensing modalities available for experimentation
- Debug interfaces enabled
- Configurable firmware parameters
- Raw data access for algorithm development
- Simulation mode for bench testing
- Data export for offline analysis

### 10.4 Configuration File

```toml
# Path 7: Research Platform

[research]
mode = true
debug_enabled = true
raw_data_export = true
simulation_mode_available = true

[nodes]
# All nodes configurable at runtime
belt = { variant = "linux_som", debug_port = "enabled" }
pendant = { variant = "curved", all_options_populated = true }
bracelet_left = { config = "all_options_populated" }
bracelet_right = { config = "all_options_populated" }
anklet_left = { config = "all_options_populated" }
anklet_right = { config = "all_options_populated" }
eyewear = { variant = "frame_integrated", all_options_populated = true }

[data]
store_raw_video = true
store_raw_audio = true
store_raw_sensor_data = true
storage_target = "sd_card"
retention_days = 0                   # Keep forever for analysis

[firmware]
debug_build = true
parameter_tuning = true
```

### 10.5 Runtime

Variable based on active configuration and experiment. Plan for external power during extended bench testing.

---

## 11. Path 8: Long-Range Detection

### 11.1 Target User

- Security personnel in open environments
- Journalists in conflict zones
- Military contractors
- Border/perimeter security

### 11.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B (Linux SoM) + piezo haptics | Full compute, fast alert |
| **Pendant** | Variant F (Long-Range Detection) | Maximize detection range |
| **Bracelets** | Standard + directional haptics | Directional alerts |
| **Anklets** | Standard | |
| **Eyewear** | Variant A (event-only) | Forward transient confirmation |

### 11.3 Long-Range Detection Specifics

**Pendant Variant F characteristics:**
- Higher-power CW radar module (not AOP)
- Optimized antenna for range
- Trade range resolution for detection distance
- Target: Detect rifle fire at 150+ meters

**Configuration:**

```toml
[extreme_velocity.detection_range]
mode = "max_range"
target_range_m = 150
range_resolution_m = 2.0           # Accept coarse resolution
velocity_range_ms = 1000          # Support up to 1000 m/s
latency_budget_ms = 5             # Relaxed; flight time is 175+ ms
```

### 11.4 Expected Performance

| Scenario | Detection Range | Flight Time | Warning Time |
|----------|-----------------|-------------|--------------|
| Rifle at 150 m | 150 m | 176 ms | 171+ ms |
| Rifle at 100 m | 150 m (detected earlier) | 117 ms | 170+ ms |
| Rifle at 50 m | 150 m (detected earlier) | 58 ms | 170+ ms |

**Key insight:** Detection at 150 m provides 170+ ms warning for all closer distances.

### 11.5 Configuration File

```toml
# Path 8: Long-Range Detection

[nodes]
belt = { variant = "linux_som", piezo_haptic = true }
pendant = { 
    variant = "long_range",
    radar_tx_power_dbm = 12,
    antenna_type = "directional",
    antenna_gain_dbi = 15
}
bracelet_left = { config = "standard", directional_haptic = true }
bracelet_right = { config = "standard", directional_haptic = true }
anklet_left = { config = "standard" }
anklet_right = { config = "standard" }
eyewear = { variant = "clip_on_event", enabled = true }

[extreme_velocity]
enabled = true
mode = "production"

[extreme_velocity.detection_range]
optimization_target = "range"
target_range_m = 150
range_resolution_m = 2.0
```

---

## 12. Path 9: Close-Range Urban

### 12.1 Target User

- Urban residents
- Convenience store workers
- Rideshare drivers
- ATM users
- High-risk pedestrian environments

### 12.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant A (MCU) + piezo haptics | Minimal cost, fast alert |
| **Pendant** | Variant E (Event-Enhanced) | Fast detection optimized |
| **Bracelets** | Standard | Basic coverage |
| **Anklets** | Optional | |
| **Eyewear** | Optional | |

### 12.3 Close-Range Scenario Reality

**For handgun at 5-15 meters:**

| Distance | Flight Time | Alert Latency | Warning Time | Human Reaction |
|----------|-------------|---------------|--------------|----------------|
| 15 m | 42 ms | 0.65-2.4 ms | 40+ ms | Insufficient for deliberate action |
| 10 m | 28 ms | 0.65-2.4 ms | 26+ ms | Insufficient for deliberate action |
| 5 m | 14 ms | 0.65-2.4 ms | 12+ ms | Awareness only |

**Honest assessment:** Close-range handgun scenarios provide awareness and startle response, not time for deliberate protective action. This is physically constrained.

### 12.4 Value Proposition for Close-Range

Despite limited warning time:
- **Immediate awareness** of threat existence
- **Direction indication** (from Tier 2)
- **Evidence capture** (Tier 3 recording)
- **Post-incident reconstruction**

### 12.5 Configuration File

```toml
# Path 9: Close-Range Urban

[nodes]
belt = { variant = "mcu", piezo_haptic = true }
pendant = { 
    variant = "event_enhanced",
    event_camera_count = 2,
    cw_radar = true
}
bracelet_left = { config = "standard" }
bracelet_right = { config = "standard" }

[extreme_velocity]
enabled = true
mode = "production"

[extreme_velocity.detection_range]
optimization_target = "latency"
target_range_m = 20
latency_budget_ms = 1

[extreme_velocity.tier1]
local_haptic_trigger = true
haptic_type = "piezo"

[data]
continuous_recording = false
recording_trigger = "on_detection"
include_integrity_chain = true
```

---

## 13. Path 10: Dual-Mode (Awareness + Detection)

### 13.1 Target User

- General consumer wanting both everyday utility and safety capability
- Balance of form factor, battery life, and capability
- "Normal wearable" that also provides detection when needed

### 13.2 Recommended Configuration

| Node | Variant/Configuration | Notes |
|------|----------------------|-------|
| **Belt** | Variant B (Linux SoM) | Full capability |
| **Pendant** | Variant B (360° Curved) + CW radar option | 360° awareness + detection |
| **Bracelets** | Standard + camera option | Full coverage |
| **Anklets** | Extended | Gait + coverage |
| **Eyewear** | Variant A (event-only) | Forward detection |

### 13.3 Dual-Mode Operation

**Everyday Mode (Sparse):**
- 360° visual awareness
- Sparse tracking
- Gait analysis
- Extended runtime (10-15 hours)

**Alert Mode (Triggered):**
- Automatic switch to dense SLAM
- Extreme velocity detection activated
- Recording starts
- Full processing engaged

### 13.4 Configuration File

```toml
# Path 10: Dual-Mode (Awareness + Detection)

[nodes]
belt = { variant = "linux_som" }
pendant_360 = { 
    variant = "curved", 
    camera_count = 6,
    camera_resolution = "720p",
    cw_radar = true,
    event_camera_count = 2
}
bracelet_left = { config = "extended", camera = true }
bracelet_right = { config = "extended", camera = true }
anklet_left = { config = "extended" }
anklet_right = { config = "extended" }
eyewear = { variant = "clip_on_event", enabled = false }

[policy]
default_mode = "PrivacyFirst"
auto_activate_on_threat = true
threat_threshold = "extreme_velocity"

[tracking]
default_mode = "sparse"
auto_switch_to_dense = true
switch_trigger = "threat_detected"

[extreme_velocity]
enabled = true
mode = "production"
standby_when_idle = true          # Power saving

[data]
continuous_recording = false
recording_trigger = "on_detection"
store_raw_video = true
include_integrity_chain = true
```

---

## 14. Decision Tree for Path Selection

```
START: What is your primary use case?
│
├── Want basic presence awareness, minimal cost, long battery
│   └── Path 1: Basic Awareness
│
├── Want full personal awareness with occasional streaming/recording
│   └── Path 2: Standard Personal Awareness
│
├── Security professional needing 360° + evidence capture
│   ├── Need 24/7 operation?
│   │   ├── Yes → Path 4: Extended Runtime
│   │   └── No → Path 3: Security Professional
│
├── Primary concern is fast-moving threats (projectiles)
│   ├── In open environment (rifle at 100m+)?
│   │   └── Path 8: Long-Range Detection
│   │
│   ├── In urban environment (handgun at 5-15m)?
│   │   └── Path 9: Close-Range Urban
│   │
│   └── Path 5: Extreme Velocity Detection (balanced)
│
├── Visually impaired user wanting spatial awareness
│   └── Path 6: Accessibility
│
├── Researcher or developer
│   └── Path 7: Research Platform
│
└── Want everyday wearable + safety when needed
    └── Path 10: Dual-Mode
```

---

## 15. Configuration Reference by Path

### Quick Configuration Matrix

| Path | Tracking | Recording | Extreme Velocity | Remote Access |
|------|----------|-----------|------------------|----------------|
| 1 — Basic | Sparse | None | Disabled | WiFi only |
| 2 — Standard | Sparse/Dense | On-trigger | Optional | WiFi |
| 3 — Security | Dense | Continuous | Enabled | WiFi + Cellular |
| 4 — Extended | Sparse | Continuous | Optional | WiFi + Cellular |
| 5 — Extreme Velocity | Sparse | On-trigger | Enabled | WiFi |
| 6 — Accessibility | Dense | On-trigger | Optional | WiFi |
| 7 — Research | Configurable | Configurable | Configurable | Configurable |
| 8 — Long-Range | Sparse | On-trigger | Max-range | WiFi + Cellular |
| 9 — Close-Range | Sparse | On-trigger | Min-latency | WiFi |
| 10 — Dual-Mode | Sparse/Dense auto | On-trigger | Standby | WiFi |

---

## 16. Cost and Complexity Comparison

### Relative Cost Matrix

| Path | Hardware Cost | Integration Complexity | Maintenance |
|------|---------------|------------------------|-------------|
| 1 — Basic | $ (Lowest) | Low | Low |
| 2 — Standard | $$ | Low-Medium | Low |
| 3 — Security | $$$$$ (Highest standard) | High | Medium |
| 4 — Extended | $$$$ | Medium | Medium (battery mgmt) |
| 5 — Extreme Velocity | $$$ | Medium | Medium |
| 6 — Accessibility | $$$$ | Medium | Medium |
| 7 — Research | Variable | High | High (experimental) |
| 8 — Long-Range | $$$$ | Medium | Medium |
| 9 — Close-Range | $$ | Low | Low |
| 10 — Dual-Mode | $$$$ | Medium | Medium |

### Complexity Factors

| Factor | Low Complexity | High Complexity |
|--------|----------------|-----------------|
| Node count | 4-5 nodes | 6+ nodes |
| Camera count | 0-1 cameras | 8+ cameras |
| UWB enabled | No | Yes |
| Cellular enabled | No | Yes |
| Custom frames | No | Yes |
| Hot-swap batteries | No | Yes |

---

## 17. Runtime Estimates by Path

### Sparse Mode (Default)

| Path | Belt Variant | Battery | Runtime |
|------|--------------|---------|---------|
| 1 — Basic | MCU | 2000 mAh | 15-20 hours |
| 2 — Standard | Linux SoM | 5000 mAh | 10-15 hours |
| 3 — Security | Linux SoM | 7000 mAh | 10-12 hours |
| 4 — Extended | Hot-swap | 7000 mAh × 2 | Unlimited |
| 5 — Extreme Velocity | Linux SoM | 5000 mAh | 10-12 hours |
| 6 — Accessibility | Linux SoM | 5000 mAh | 10-15 hours |
| 7 — Research | Variable | Variable | Variable |
| 8 — Long-Range | Linux SoM | 5000 mAh | 10-12 hours |
| 9 — Close-Range | MCU | 2000 mAh | 15-20 hours |
| 10 — Dual-Mode | Linux SoM | 5000 mAh | 10-15 hours |

### Full Active Mode (SLAM + Streaming)

| Path | Runtime |
|------|---------|
| 2 — Standard | 5-8 hours |
| 3 — Security | 5-8 hours |
| 4 — Extended | Unlimited (with swaps) |
| 5 — Extreme Velocity | 8-10 hours |
| 6 — Accessibility | 5-8 hours |
| 10 — Dual-Mode | 5-8 hours |

---

## 18. Connectivity Architecture for All Paths

### Universal Constraint

**Only the Belt Node connects to external networks.** This applies to all paths.

```
All Nodes (Pendant, Bracelets, Anklets, Eyewear)
       │
       │  BAN (BLE 5.x / UWB)
       ▼
  Belt Node (Sole Gateway)
       │
       ├── WiFi ──► Companion App (local)
       ├── Cellular ──► Remote Access
       └── BLE ──► Direct App (fallback)
```

### BAN Configuration by Path

| Path | BLE Mode | UWB | Notes |
|------|----------|-----|-------|
| 1 — Basic | Standard | No | Basic scheduling |
| 2 — Standard | Standard | Optional | Normal intervals |
| 3 — Security | Optimized | Recommended | QoS classes, antenna diversity |
| 4 — Extended | Power-saving | Optional | Longer intervals when idle |
| 5 — Extreme Velocity | Optimized + QoS Critical | Recommended | Reserved slots |
| 6 — Accessibility | Standard | Optional | |
| 7 — Research | Configurable | Configurable | |
| 8 — Long-Range | Optimized | Recommended | |
| 9 — Close-Range | Optimized | Optional | QoS Critical |
| 10 — Dual-Mode | Standard + auto-optimize | Optional | |

---

## 19. Upgrade Paths

### Starting from Path 1 (Basic Awareness)

| Upgrade To | Required Changes |
|------------|------------------|
| Path 2 | Belt MCU → Linux SoM; add pendant camera |
| Path 5 | Add event-enhanced pendant; enable extreme velocity |
| Path 9 | Add event-enhanced pendant; enable extreme velocity |

### Starting from Path 2 (Standard Personal)

| Upgrade To | Required Changes |
|------------|------------------|
| Path 3 | Pendant → 360° Curved; add more cameras; cellular |
| Path 5 | Add event cameras; enable extreme velocity |
| Path 10 | Add CW radar to pendant; enable extreme velocity |

### Starting from Path 5 (Extreme Velocity)

| Upgrade To | Required Changes |
|------------|------------------|
| Path 8 | Pendant → Long-Range variant; optimize for range |
| Path 10 | Pendant → 360° Curved; add SLAM |

### Modular Upgrade Philosophy

**Many upgrades are configuration changes, not hardware replacement:**
- Enabling extreme velocity detection: firmware configuration
- Adding remote access: cellular module installation
- Changing alert mode: software configuration
- Adjusting detection range: firmware parameters

**Some upgrades require hardware:**
- Belt MCU → Linux SoM: PCB replacement
- Pendant standard → 360° Curved: PCB replacement
- Adding UWB: Module installation (if footprint present)

---

## 20. Legal and Compliance Considerations

### Recording Consent

**User responsibility:** Recording laws vary by jurisdiction. Users must ensure compliance with:
- One-party vs. two-party consent
- Public vs. private space distinctions
- Workplace surveillance regulations
- Residential surveillance disclosure requirements

### Data Handling

**All paths support user-configured data handling:**

```toml
[data]
# User controls all retention and storage
retention_days = 30              # Or 0 = keep forever
storage_target = "sd_card"       # Local only by default
include_integrity_chain = true   # For legal evidence
```

### Disclaimers

**For all paths:**
- Not a certified safety system
- Not personal protective equipment
- Not a medical device
- Detection capability ≠ protection capability
- User assumes all responsibility for deployment

**For extreme velocity paths (5, 8, 9, 10):**
- Detection provides awareness, not physical protection
- Warning times at close range are physically limited
- No capability to prevent impact from detected projectiles

### Export Control

- Standard sensing configurations: No export restrictions
- Cellular modules: Subject to standard telecommunications regulations
- No weapons-related technology in any path

---

## Appendix A: Node Count by Path

| Path | Pendant | Bracelets | Belt | Anklets | Eyewear | Total |
|------|---------|-----------|------|---------|---------|-------|
| 1 — Basic | 1 | 2 | 1 | 2 | 0 | 6 |
| 2 — Standard | 1 | 2 | 1 | 2 | 0-1 | 6-7 |
| 3 — Security | 1 (360°) | 2 | 1 | 2 | 1 | 7 |
| 4 — Extended | 1 | 2 | 1 | 2 | 0-1 | 6-7 |
| 5 — Extreme Velocity | 1 | 2 | 1 | 2 | 1 | 7 |
| 6 — Accessibility | 1 (360°) | 2 | 1 | 2 | 0 | 6 |
| 7 — Research | 1-2 | 2 | 1 | 2 | 0-1 | 6-8 |
| 8 — Long-Range | 1 | 2 | 1 | 2 | 1 | 7 |
| 9 — Close-Range | 1 | 2 | 1 | 0-2 | 0-1 | 4-7 |
| 10 — Dual-Mode | 1 (360°) | 2 | 1 | 2 | 1 | 7 |

---

## Appendix B: Quick Reference Cards

### Path 1 — Basic Awareness Card

```
┌─────────────────────────────────────┐
│     PATH 1: BASIC AWARENESS         │
├─────────────────────────────────────┤
│ Belt:     MCU or ESP32              │
│ Pendant:  Standard, no camera       │
│ Tracking: Sparse only               │
│ Recording: None                     │
│ Runtime:  15-20 hours               │
│ Cost:     Lowest                    │
│ Use Case: Minimal presence awareness│
└─────────────────────────────────────┘
```

### Path 3 — Security Professional Card

```
┌─────────────────────────────────────┐
│   PATH 3: SECURITY PROFESSIONAL     │
├─────────────────────────────────────┤
│ Belt:     Linux SoM + piezo         │
│ Pendant:  360° Curved (8 cameras)   │
│ Tracking: Dense SLAM                │
│ Recording: Continuous/triggered     │
│ Runtime:  5-8 hours (active)        │
│ Cost:     Highest standard          │
│ Use Case: 360° + evidence capture   │
└─────────────────────────────────────┘
```

### Path 5 — Extreme Velocity Detection Card

```
┌─────────────────────────────────────┐
│  PATH 5: EXTREME VELOCITY DETECTION │
├─────────────────────────────────────┤
│ Belt:     Linux SoM + piezo         │
│ Pendant:  Event-Enhanced + CW radar │
│ Detection: Tiered (reflex → char)   │
│ Warning:  Rifle @ 100m: 115+ ms     │
│ Runtime:  8-12 hours                │
│ Cost:     Moderate                  │
│ Use Case: Fast threat detection     │
└─────────────────────────────────────┘
```

---

**End of Deployment Paths Document**
