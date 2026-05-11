# Power Profiles — SENTINEL-WEAR

**Version:** 1.0
**Status:** Comprehensive Reference
**Applies to:** All Node Variants and Deployment Paths

---

## Table of Contents

1. [Overview and Philosophy](#1-overview-and-philosophy)
2. [Power Management Architecture](#2-power-management-architecture)
3. [Belt Node Power Profiles](#3-belt-node-power-profiles)
4. [Pendant Node Power Profiles](#4-pendant-node-power-profiles)
5. [360° Curved Pendant Power Profiles](#5-360-curved-pendant-power-profiles)
6. [Bracelet Node Power Profiles](#6-bracelet-node-power-profiles)
7. [Anklet Node Power Profiles](#7-anklet-node-power-profiles)
8. [Eyewear Node Power Profiles](#8-eyewear-node-power-profiles)
9. [Complete System Power Budgets](#9-complete-system-power-budgets)
10. [Operational Modes and Power States](#10-operational-modes-and-power-states)
11. [Battery Sizing and Runtime Calculations](#11-battery-sizing-and-runtime-calculations)
12. [Thermal Management](#12-thermal-management)
13. [Deployment Path Power Analysis](#13-deployment-path-power-analysis)
14. [Extreme Velocity Detection Power Requirements](#14-extreme-velocity-detection-power-requirements)
15. [Connectivity Power Impact](#15-connectivity-power-impact)
16. [SLAM and Dense World Model Power Requirements](#16-slam-and-dense-world-model-power-requirements)
17. [Power Optimization Strategies](#17-power-optimization-strategies)
18. [Configuration Reference](#18-configuration-reference)
19. [Charging Architecture](#19-charging-architecture)
20. [Monitoring and Alerts](#20-monitoring-and-alerts)

---

## 1. Overview and Philosophy

### 1.1 The Fundamental Challenge

SENTINEL-WEAR operates under severe power constraints that fundamentally shape its architecture:

| Constraint | Impact |
|------------|--------|
| Jewelry form factor | Limited battery volume (100-1500 mAh typical) |
| Skin contact | Maximum 40-42°C temperature |
| Continuous operation | 12-24+ hour runtime required |
| Multiple radios | BLE, optional UWB, WiFi (belt only) |
| Compute-intensive features | SLAM, fusion, inference |

**These constraints are why the architecture exists in its current form.** The belt node as sole external gateway, the BLE-centric BAN, the tiered detection architecture — all flow from the power budget.

### 1.2 Core Philosophy: No Artificial Limits

Power management is user-configurable. The system exposes all power-related parameters and allows users to make tradeoffs according to their priorities:

- Maximum battery life vs. maximum capability
- Continuous SLAM vs. on-demand SLAM
- Always-on extreme velocity detection vs. triggered activation
- High-bandwidth streaming vs. metadata-only

**The system does not impose artificial power limits.** It provides data, exposes parameters, and lets users configure their power profile.

### 1.3 The Power Budget Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         POWER BUDGET HIERARCHY                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TIER 1 — ALWAYS-ON SENSING (Highest Priority, Always Active)               │
│  ├── BLE BAN communication (all nodes)                                      │
│  ├── IMU sampling (all nodes)                                               │
│  ├── mmWave radar presence detection (pendant, bracelets, belt)             │
│  └── Power: 50-200 mW per node                                              │
│                                                                              │
│  TIER 2 — EVENT-TRIGGERED HIGH-FIDELITY SENSING                              │
│  ├── LiDAR/ToF activation (anklets, pendant)                                │
│  ├── Acoustic processing (pendant)                                          │
│  ├── Event camera monitoring (pendant, eyewear)                             │
│  └── Power: 200-800 mW additional per node                                  │
│                                                                              │
│  TIER 3 — CONTINUOUS HIGH-FIDELITY CAPTURE                                   │
│  ├── 360° camera streaming (360° pendant)                                   │
│  ├── SLAM processing (belt node)                                            │
│  ├── Continuous video recording                                             │
│  └── Power: 2-5 W additional                                                │
│                                                                              │
│  TIER 4 — EXTERNAL CONNECTIVITY                                              │
│  ├── WiFi streaming (belt node)                                             │
│  ├── Cellular connectivity (belt node)                                      │
│  └── Power: 300-1200 mW additional                                          │
│                                                                              │
│  TIER 5 — EXTREME VELOCITY DETECTION                                         │
│  ├── CW radar continuous monitoring                                         │
│  ├── Event camera fast transient detection                                  │
│  └── Power: 150-400 mW additional                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Power Management Architecture

### 2.1 Power Domains

**All nodes share a common power domain structure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    NODE POWER DOMAINS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VBAT (3.7-4.2V)                                                │
│  ├── Raw battery voltage                                         │
│  ├── Powers: BLE radio PA, haptic peak, camera peak              │
│  └── Current capability: Direct from cell (up to 2A pulse)      │
│                                                                  │
│  VCC_3V3 (3.3V)                                                 │
│  ├── Main regulated rail                                        │
│  ├── Powers: MCU, digital sensors, BLE radio core               │
│  └── Source: Buck converter from VBAT (90%+ efficiency)         │
│                                                                  │
│  VCC_1V8 (1.8V)                                                 │
│  ├── Low-power rail                                             │
│  ├── Powers: IMU (low-power mode), environmental sensors         │
│  └── Source: LDO from VCC_3V3 (low noise)                       │
│                                                                  │
│  V_CAM / V_SENSOR (2.8V / 3.3V, optionally gated)               │
│  ├── Camera and sensor power                                     │
│  ├── Optionally gated by hardware switch or MCU GPIO             │
│  └── Source: Buck or LDO from VBAT                              │
│                                                                  │
│  V_RADIO (3.3V)                                                 │
│  ├── BLE/UWB radio power                                         │
│  ├── Always-on for BAN communication                            │
│  └── Source: Buck from VBAT                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Belt Node Additional Power Domains

```
┌─────────────────────────────────────────────────────────────────┐
│                    BELT NODE ADDITIONAL DOMAINS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  V_CELLULAR (3.3V / 4.2V)                                       │
│  ├── Cellular module power                                       │
│  ├── High current capability (up to 2A peak)                    │
│  └── Source: Dedicated buck from main battery                   │
│                                                                  │
│  V_WIFI (3.3V)                                                  │
│  ├── WiFi module power (if external to SoM)                      │
│  └── Source: Buck from main battery                             │
│                                                                  │
│  V_PIEXO (5V)                                                   │
│  ├── Piezo haptic driver boost output                           │
│  └── Source: Boost converter (50-200 Vpp)                       │
│                                                                  │
│  V_EXT (5V)                                                      │
│  ├── External power output (for pendant extended runtime)        │
│  └── Source: Buck from main battery                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Power Management ICs

**Recommended components per node class:**

| Node Class | PMIC Recommendation | Features |
|------------|---------------------|----------|
| Pendant Standard | Nordic nPM1300 | Integrated charger, fuel gauge, dual buck |
| Pendant 360° | TI BQ25895 + TPS65217 | Higher current, multi-rail |
| Medallion | TI BQ25895 | 5A charge, multi-rail |
| Bracelet | Nordic nPM1300 | Compact, integrated |
| Belt (MCU) | BQ25895 + multiple bucks | High current, dual battery support |
| Belt (Linux SoM) | Dedicated PMIC per SoM + BQ25895 | Platform-specific |
| Anklet | Nordic nPM1300 or generic | Standard |
| Eyewear | TP4056 + XC6206 | Minimal, low cost |

---

## 3. Belt Node Power Profiles

### 3.1 Variant A — MCU-Class (Minimal)

**Hardware configuration:**
- MCU: STM32H743 or equivalent (Cortex-M7, 480 MHz)
- No Linux, embedded RTOS
- BLE 5.x (integrated or module)
- Optional UWB module
- Optional cellular module

**Power breakdown by state:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU (H7 core) | 5 mW | 50 mW | 150 mW | 400 mW |
| BLE radio (advertising) | 5 mW | 15 mW | 50 mW | 100 mW |
| BLE radio (connected) | — | 20 mW | 40 mW | 80 mW |
| UWB (if active) | — | 50 mW | 100 mW | 150 mW |
| mmWave radar (duty-cycled) | — | 50 mW | 200 mW | 400 mW |
| IMU (continuous) | 1 mW | 5 mW | 10 mW | 20 mW |
| Environmental | — | 2 mW | 5 mW | 10 mW |
| Cellular idle | — | 30 mW | — | — |
| Cellular active | — | — | 300 mW | 600 mW |

**Total power by operational state:**

| State | Power | Description |
|-------|-------|-------------|
| Deep Sleep | 10-20 mW | MCU in low-power mode, BLE advertising |
| Sleep | 30-50 mW | BAN idle, sensors duty-cycled |
| Sparse Active | 150-300 mW | Full BAN, sparse tracking, no SLAM |
| Full Active | 300-600 mW | All sensing active, streaming |
| Peak (cellular + all sensors) | 600-1200 mW | Maximum capability |

**Battery sizing:**

| Battery | Deep Sleep | Sleep | Sparse Active | Full Active |
|---------|------------|-------|---------------|-------------|
| 2000 mAh | 333+ hours | 100+ hours | 15-20 hours | 5-8 hours |
| 3000 mAh | 500+ hours | 150+ hours | 20-30 hours | 7-12 hours |

**Recommended configuration:**
- 2000 mAh minimum for 12+ hour sparse operation
- No SLAM capability (MCU-class cannot run SLAM)
- Suitable for basic presence awareness

### 3.2 Variant B — Linux SoM (Standard)

**Hardware configuration:**
- SoM: Raspberry Pi CM4 or NXP i.MX 8M Plus
- Full Linux, all crates operational
- WiFi (integrated)
- BLE (via co-processor or USB)
- Optional UWB
- Optional cellular module

**Power breakdown by component:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| SoM (Linux idle) | 200 mW | 500 mW | 1500 mW | 3000 mW |
| WiFi (idle) | 50 mW | 100 mW | 300 mW | 500 mW |
| WiFi (streaming) | — | — | 400 mW | 600 mW |
| BLE radio | — | 20 mW | 50 mW | 100 mW |
| UWB (if active) | — | 50 mW | 100 mW | 150 mW |
| mmWave radar | — | 50 mW | 200 mW | 400 mW |
| IMU | 1 mW | 5 mW | 10 mW | 20 mW |
| Environmental | — | 2 mW | 5 mW | 10 mW |
| Cellular idle | — | 30 mW | — | — |
| Cellular (LTE Cat 4 active) | — | — | 400 mW | 700 mW |
| Cellular (5G active) | — | — | 800 mW | 1200 mW |

**Total power by operational state:**

| State | Power | Description |
|-------|-------|-------------|
| Deep Sleep | 50-100 mW | Linux suspend, BLE idle |
| Sleep | 200-400 mW | Linux idle, sensors duty-cycled |
| Sparse Active | 600-1000 mW | Full BAN, sparse tracking, WiFi idle |
| Dense Active | 2000-3500 mW | SLAM active, sparse streaming |
| Full Active | 3500-5000 mW | SLAM + streaming + all sensors |
| Peak (5G + SLAM + streaming) | 5000-7000 mW | Maximum capability |

**Battery sizing:**

| Battery | Deep Sleep | Sleep | Sparse Active | Dense Active | Full Active |
|---------|------------|-------|---------------|--------------|-------------|
| 5000 mAh | 100+ hours | 25-40 hours | 10-16 hours | 4-6 hours | 2-3 hours |
| 7000 mAh | 140+ hours | 35-55 hours | 14-22 hours | 5-8 hours | 3-4 hours |

**Recommended configuration:**
- 5000 mAh for standard deployment (10+ hour sparse operation)
- 7000 mAh for extended sparse operation (15+ hours)
- Default to sparse mode; dense/full active on-demand only

### 3.3 Variant C — High-Performance

**Hardware configuration:**
- SoM: Qualcomm SA8155P or NXP i.MX 8M Plus with NPU
- GPU-class compute
- Neural inference capability
- Full SLAM acceleration

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| SoM + GPU | 300 mW | 800 mW | 3000 mW | 6000 mW |
| NPU active | — | — | 500 mW | 1500 mW |
| WiFi | 50 mW | 100 mW | 400 mW | 600 mW |
| BLE | — | 20 mW | 50 mW | 100 mW |
| UWB | — | 50 mW | 100 mW | 150 mW |
| Radar + IMU | — | 60 mW | 220 mW | 420 mW |
| Cellular (5G) | — | 30 mW | 800 mW | 1200 mW |

**Total power by operational state:**

| State | Power | Description |
|-------|-------|-------------|
| Deep Sleep | 100-200 mW | Linux suspend |
| Sleep | 400-700 mW | Linux idle |
| Sparse Active | 1000-1500 mW | Full BAN, sparse tracking |
| Dense Active | 3500-5000 mW | SLAM + NPU inference |
| Full Active | 5000-8000 mW | All compute + streaming |
| Peak | 8000-12000 mW | Maximum (rare) |

**Battery sizing:**

| Battery | Deep Sleep | Sleep | Sparse Active | Dense Active | Full Active |
|---------|------------|-------|---------------|--------------|-------------|
| 7000 mAh | 70+ hours | 20-35 hours | 8-12 hours | 2-3 hours | 1-2 hours |
| 10000 mAh | 100+ hours | 28-50 hours | 11-17 hours | 3-4 hours | 1.5-2.5 hours |

**Recommended configuration:**
- 7000 mAh minimum
- Default to sparse mode for all-day use
- Dense mode for short-duration high-value events
- Best for: production deployment, maximum capability, professional use

### 3.4 Variant D — ESP32-Based

**Hardware configuration:**
- MCU: ESP32-S3
- Integrated WiFi + BLE
- Lowest cost
- Limited compute (no SLAM)

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| ESP32-S3 (sleep) | 5 mW | — | — | — |
| ESP32-S3 (active) | — | 50 mW | 150 mW | 400 mW |
| WiFi (idle) | — | 30 mW | — | — |
| WiFi (active) | — | — | 200 mW | 400 mW |
| BLE (connected) | — | 20 mW | 40 mW | 80 mW |
| Radar + IMU | — | 60 mW | 220 mW | 420 mW |

**Total power:**

| State | Power |
|-------|-------|
| Deep Sleep | 5-10 mW |
| Sleep | 50-80 mW |
| Sparse Active | 200-400 mW |
| Full Active | 400-800 mW |

**Battery sizing:**

| Battery | Deep Sleep | Sleep | Sparse Active | Full Active |
|---------|------------|-------|---------------|-------------|
| 2000 mAh | 500+ hours | 50-80 hours | 10-20 hours | 5-10 hours |

**Recommended configuration:**
- Minimal deployment
- No SLAM
- WiFi as BAN transport to external compute (companion app or edge device)
- Suitable for: research prototypes, minimal cost deployments

### 3.5 Variant E — Extended Runtime / Hot-Swappable

**Hardware configuration:**
- Any compute variant (A, B, or C)
- Dual battery bays
- Power path management for hot-swap
- Larger enclosure

**Power architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOT-SWAP POWER ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Battery Bay 1 ──┐                                              │
│                   ├── Power Path Controller ──► System Load    │
│  Battery Bay 2 ──┘                                              │
│                                                                  │
│  Operation:                                                      │
│  ├── Normal: Bay 1 powers system, Bay 2 charging                │
│  ├── Swap: Bay 2 powers system while Bay 1 swapped              │
│  └── No downtime during swap                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Power budget:**

| Configuration | Single Battery | Dual Battery | Continuous Operation |
|---------------|----------------|--------------|----------------------|
| Sparse Active | 600-1000 mW | 1200-2000 mW total | Unlimited (with swaps) |
| Dense Active | 2500-4000 mW | 5000-8000 mW total | Unlimited (with swaps) |
| Full Active | 4000-7000 mW | 8000-14000 mW total | Unlimited (with swaps) |

**Swap timing:**
- Battery depletion warning at 15%
- User has ~20-30 minutes to swap before complete depletion
- Swap operation: 5-10 seconds
- No data loss, no service interruption

**Recommended configuration:**
- Professional security, 24/7 operation
- One battery charging while other in use
- Carried spare batteries for extended duration

---

## 4. Pendant Node Power Profiles

### 4.1 Variant A — Standard Flat

**Hardware configuration:**
- MCU: nRF5340 or STM32WB55
- mmWave radar (Acconeer XR112 or TI IWR6843)
- IMU (BMI270)
- Microphone array (3-4 elements)
- Optional camera
- Battery: 150-300 mAh

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU (sleep) | 1 mW | — | — | — |
| MCU (active) | — | 20 mW | 50 mW | 100 mW |
| BLE (advertising) | 2 mW | 5 mW | 15 mW | 30 mW |
| BLE (connected) | — | 10 mW | 20 mW | 40 mW |
| mmWave radar (duty) | — | 20 mW | 100 mW | 200 mW |
| IMU | 1 mW | 3 mW | 5 mW | 10 mW |
| Microphone array | — | 10 mW | 30 mW | 50 mW |
| Camera (if active) | — | 50 mW | 150 mW | 300 mW |

**Total power:**

| State | Power | Battery Life (200 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 3-5 mW | 50-80 hours |
| Sleep | 30-50 mW | 8-13 hours |
| Sparse Active (no camera) | 100-200 mW | 2-4 hours |
| Full Active (with camera) | 200-400 mW | 1-2 hours |

**Recommended configuration:**
- Default to sparse sensing with duty-cycled radar
- Camera activation on trigger only
- 24-48 hour runtime achievable with sparse mode

### 4.2 Variant C — Medallion (Premium)

**Hardware configuration:**
- Larger enclosure (60 mm diameter)
- Higher-capacity battery (800-1500 mAh)
- Optional higher-resolution camera
- More microphones (6-element)

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU | 1 mW | 30 mW | 80 mW | 150 mW |
| BLE | — | 10 mW | 25 mW | 50 mW |
| mmWave radar | — | 30 mW | 150 mW | 300 mW |
| IMU | 1 mW | 5 mW | 10 mW | 20 mW |
| 6-element mic array | — | 20 mW | 60 mW | 100 mW |
| Higher-res camera | — | 100 mW | 300 mW | 600 mW |

**Total power:**

| State | Power | Battery Life (1000 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 5-10 mW | 100-200 hours |
| Sleep | 70-120 mW | 17-28 hours |
| Sparse Active | 200-400 mW | 5-10 hours |
| Full Active | 500-900 mW | 2-4 hours |

**Recommended configuration:**
- All-day sparse operation with premium sensing
- Camera on trigger or schedule
- Suitable for: professional deployment, extended capability

### 4.3 Variant D — Tactical / Extended Runtime

**Hardware configuration:**
- Larger profile (70-80 mm)
- High-capacity battery (up to 2000 mAh)
- Optional belt power input

**Power breakdown:**

| State | Power | Battery Life (2000 mAh) | With Belt Power |
|-------|-------|------------------------|------------------|
| Deep Sleep | 5-10 mW | 200-400 hours | Unlimited |
| Sleep | 70-120 mW | 33-57 hours | Unlimited |
| Sparse Active | 200-400 mW | 10-20 hours | Unlimited |
| Full Active | 500-900 mW | 4-8 hours | Unlimited |

**Belt power input:**
- When connected, pendant draws from belt's larger battery
- Extends runtime to match belt node runtime
- Useful for: 360° pendant continuous operation

### 4.4 Variant E — Event-Enhanced

**Hardware configuration:**
- Standard pendant sensors
- Plus dedicated event cameras (2)
- CW radar module

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| All standard pendant | 5-10 mW | 50-100 mW | 200-400 mW | 500 mW |
| Event cameras (2) | — | 40 mW | 80 mW | 150 mW |
| CW radar (continuous) | — | 50 mW | 100 mW | 150 mW |

**Total power:**

| State | Power | Battery Life (300 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 5-10 mW | 30-60 hours |
| Sleep | 100-200 mW | 3-6 hours |
| Sparse Active | 300-500 mW | 1.2-2 hours |
| Full Active + Extreme Velocity | 500-800 mW | 0.75-1.2 hours |

**Recommended configuration:**
- CW radar continuous for extreme velocity detection
- Event cameras on standby
- Higher battery drain for detection capability
- Use belt power input for extended operation

### 4.5 Variant F — Long-Range Detection

**Hardware configuration:**
- Higher-power CW radar module
- Optimized antenna for range
- Standard event camera

**Power breakdown:**

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU | 1 mW | 30 mW | 80 mW | 150 mW |
| High-power CW radar | — | 100 mW | 200 mW | 400 mW |
| Event camera | — | 20 mW | 50 mW | 100 mW |
| BLE | — | 10 mW | 25 mW | 50 mW |

**Total power:**

| State | Power | Battery Life (400 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 5-10 mW | 40-80 hours |
| Sleep | 150-200 mW | 4-5 hours |
| Active (detection mode) | 300-500 mW | 1.6-2.7 hours |

**Recommended configuration:**
- Use belt power input for continuous operation
- Detection mode activates on movement or user trigger
- Battery backup for cordless operation

---

## 5. 360° Curved Pendant Power Profiles

### 5.1 Overview

The 360° curved pendant is a **power-intensive node** requiring careful thermal and battery management. It is the most capable visual sensing node but has the highest power draw.

### 5.2 Component Power Breakdown

| Component | Quantity | Power Each | Total Power |
|-----------|----------|------------|-------------|
| Camera modules | 4-8 | 100-200 mW | 400-1600 mW |
| Vision processor | 1 | 500-1500 mW | 500-1500 mW |
| mmWave radar | 1-2 | 100-200 mW | 100-400 mW |
| IMU array | 1-8 | 5 mW | 5-40 mW |
| Microphone array | 4-8 | 10 mW | 40-80 mW |
| BLE/UWB radio | 1 | 50-150 mW | 50-150 mW |
| **Total Active** | — | — | **1100-3770 mW** |

### 5.3 Power by Configuration

**Minimal Configuration (4 cameras, VGA quality):**

| State | Power | Battery Life (600 mAh) |
|-------|-------|------------------------|
| Sleep | 50-100 mW | 12-24 hours |
| Sparse Active (no cameras) | 200-400 mW | 3-6 hours |
| Cameras Active (VGA) | 1500-2000 mW | 0.6-0.8 hours |
| Full Active (all features) | 2000-2500 mW | 0.5-0.6 hours |

**Standard Configuration (8 cameras, 720p quality):**

| State | Power | Battery Life (800 mAh) |
|-------|-------|------------------------|
| Sleep | 50-100 mW | 16-32 hours |
| Sparse Active (no cameras) | 250-500 mW | 3-6 hours |
| Cameras Active (720p) | 2500-3500 mW | 0.5-0.65 hours |
| Full Active (streaming) | 3500-4500 mW | 0.35-0.45 hours |

**Maximum Configuration (8 cameras, 1080p, streaming):**

| State | Power | Battery Life (1000 mAh) |
|-------|-------|------------------------|
| Sleep | 50-100 mW | 20-40 hours |
| Sparse Active | 300-600 mW | 3-6 hours |
| Cameras Active (1080p) | 3500-5000 mW | 0.4-0.57 hours |
| Full Streaming | 4500-6000 mW | 0.33-0.44 hours |

### 5.4 Thermal Constraints

**Maximum sustained power for jewelry form factor:**
- 50×50×15 mm pendant: ~2W sustained before exceeding 42°C
- 80×60×20 mm pendant: ~3W sustained
- Belt power input required for higher sustained operation

**Duty cycling strategy:**

```toml
[pendant_360.power]
# Thermal management
max_sustained_power_mw = 2500     # Limit to prevent overheating
thermal_throttle_enabled = true
thermal_throttle_threshold_c = 40
thermal_throttle_action = "reduce_camera_fps"

# Camera duty cycling for extended operation
camera_duty_cycle_enabled = true
camera_active_duration_ms = 5000   # 5 seconds active
camera_idle_duration_ms = 15000    # 15 seconds idle
effective_fps_reduction = 0.25     # 25% of full-time operation
```

### 5.5 Extended Runtime Strategies

**Strategy 1: Progressive Quality**

```toml
[pendant_360.progressive_quality]
enabled = true
# Start at lower quality, improve over time
initial_quality = "qvga"          # 320×240, very low power
target_quality = "720p"           # Target after warmup
warmup_duration_s = 10            # 10 seconds to reach target

# Power profile over time
# Time 0-5s:   QVGA baseline (~500 mW)
# Time 5-10s:  VGA progressive (~1000 mW)
# Time 10s+:   720p full quality (~2500 mW)
```

**Strategy 2: Event-Triggered High Quality**

```toml
[pendant_360.event_triggered]
enabled = true
# Low quality when nothing happening
idle_quality = "qvga"
idle_power_mw = 500

# High quality on detection event
event_quality = "720p"
event_power_mw = 2500
event_duration_s = 30             # Return to idle after 30s of no events
```

**Strategy 3: Belt Power Input**

```toml
[pendant_360.belt_power]
enabled = true
# When connected to belt power, no battery constraint
# Full operation possible indefinitely
```

---

## 6. Bracelet Node Power Profiles

### 6.1 Power Breakdown

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU (nRF5340) | 0.5 mW | 10 mW | 30 mW | 60 mW |
| BLE (connected) | — | 8 mW | 15 mW | 30 mW |
| mmWave radar (duty) | — | 15 mW | 50 mW | 100 mW |
| IMU | 0.5 mW | 2 mW | 5 mW | 10 mW |
| Haptic (LRA) | — | — | 50 mW | 200 mW (pulse) |
| Camera (if equipped) | — | 40 mW | 100 mW | 200 mW |
| UWB (if equipped) | — | 30 mW | 80 mW | 150 mW |

### 6.2 Power by Configuration

**Minimal Configuration (no camera, no UWB):**

| State | Power | Battery Life (150 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 75-150 hours |
| Sleep | 20-30 mW | 10-15 hours |
| Sparse Active | 50-100 mW | 3-6 hours |
| Full Active + Haptic | 100-200 mW | 1.5-3 hours |

**Standard Configuration (no camera, optional UWB):**

| State | Power | Battery Life (250 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 125-250 hours |
| Sleep | 25-40 mW | 12-20 hours |
| Sparse Active | 80-150 mW | 3-6 hours |
| Full Active + UWB | 150-300 mW | 1.7-3.3 hours |

**Extended Configuration (with camera):**

| State | Power | Battery Life (400 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 200-400 hours |
| Sleep | 30-50 mW | 16-27 hours |
| Sparse Active | 100-200 mW | 4-8 hours |
| Full Active + Camera | 250-400 mW | 2-3 hours |

---

## 7. Anklet Node Power Profiles

### 7.1 Power Breakdown

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU | 0.5 mW | 10 mW | 30 mW | 60 mW |
| BLE | — | 8 mW | 15 mW | 30 mW |
| ToF / LiDAR | — | 20 mW | 50 mW | 100 mW |
| mmWave (extended) | — | 15 mW | 50 mW | 100 mW |
| IMU (high-rate) | 1 mW | 5 mW | 10 mW | 20 mW |
| Haptic | — | — | 50 mW | 200 mW |
| Camera (if equipped) | — | 30 mW | 80 mW | 150 mW |
| UWB (recommended) | — | 30 mW | 80 mW | 150 mW |

### 7.2 Power by Configuration

**Minimal Configuration (ToF only):**

| State | Power | Battery Life (300 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 150-300 hours |
| Sleep | 20-30 mW | 20-30 hours |
| Active (gait monitoring) | 80-150 mW | 4-7.5 hours |

**Standard Configuration (ToF + optional mmWave):**

| State | Power | Battery Life (400 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 200-400 hours |
| Sleep | 25-40 mW | 20-32 hours |
| Active (full gait) | 120-220 mW | 3.6-6.6 hours |

**Extended Configuration (with camera + UWB):**

| State | Power | Battery Life (500 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1-2 mW | 250-500 hours |
| Sleep | 35-60 mW | 17-29 hours |
| Active | 200-400 mW | 2.5-5 hours |
| Full Active + Camera | 350-600 mW | 1.7-2.9 hours |

---

## 8. Eyewear Node Power Profiles

### 8.1 Power Breakdown

| Component | Sleep | Idle | Active | Peak |
|-----------|-------|------|--------|------|
| MCU | 0.5 mW | 5 mW | 15 mW | 30 mW |
| BLE | — | 5 mW | 10 mW | 20 mW |
| Event camera | — | 10 mW | 30 mW | 60 mW |
| Conventional camera | — | 30 mW | 80 mW | 150 mW |
| IMU | 0.5 mW | 2 mW | 5 mW | 10 mW |

### 8.2 Power by Configuration

**Clip-on (Variant A, event-only):**

| State | Power | Battery Life (50 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1 mW | 50 hours |
| Sleep | 10-15 mW | 6.7-10 hours |
| Active (event camera) | 40-70 mW | 1.4-2.5 hours |

**Clip-on (Variant B, event + camera):**

| State | Power | Battery Life (80 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1 mW | 80 hours |
| Sleep | 15-25 mW | 6.4-10.7 hours |
| Active | 80-150 mW | 1.1-2 hours |

**Frame-integrated (Variant C, full array):**

| State | Power | Battery Life (150 mAh) |
|-------|-------|------------------------|
| Deep Sleep | 1 mW | 150 hours |
| Sleep | 25-40 mW | 7.5-12 hours |
| Active | 150-300 mW | 1-2 hours |
| Full (multiple cameras) | 250-400 mW | 0.75-1.2 hours |

### 8.3 Belt Power Cable Option

**For Variant C with belt power cable:**

| Mode | Power Source | Runtime |
|------|--------------|---------|
| Battery only | 150 mAh internal | 1-2 hours active |
| Belt power cable | Belt battery | Unlimited |

---

## 9. Complete System Power Budgets

### 9.1 Minimal Deployment (Path 1)

**Configuration:**
- Belt: Variant A (MCU)
- Pendant: Variant A, no camera
- Bracelets (×2): Minimal
- Anklets (×2): Minimal
- No eyewear

**Power breakdown:**

| Node | State | Power |
|------|-------|-------|
| Belt | Sparse Active | 200-400 mW |
| Pendant | Sparse Active | 100-200 mW |
| Bracelet L | Sparse Active | 50-100 mW |
| Bracelet R | Sparse Active | 50-100 mW |
| Anklet L | Active | 80-150 mW |
| Anklet R | Active | 80-150 mW |
| **Total** | | **560-1100 mW** |

**Runtime with belt 2000 mAh battery:**
- Sparse mode: 4-7 hours

### 9.2 Standard Deployment (Path 2)

**Configuration:**
- Belt: Variant B (Linux SoM)
- Pendant: Variant A with camera
- Bracelets (×2): Standard
- Anklets (×2): Extended
- Eyewear: Optional

**Power breakdown (sparse mode):**

| Node | Power |
|------|-------|
| Belt (sparse) | 600-1000 mW |
| Pendant (sparse) | 100-200 mW |
| Bracelets (×2) | 100-200 mW |
| Anklets (×2) | 200-400 mW |
| **Total** | **1000-1800 mW** |

**Power breakdown (dense mode, streaming):**

| Node | Power |
|------|-------|
| Belt (SLAM + streaming) | 3000-5000 mW |
| Pendant (camera active) | 200-400 mW |
| Bracelets (×2) | 100-200 mW |
| Anklets (×2) | 200-400 mW |
| **Total** | **3500-6000 mW** |

**Runtime with belt 5000 mAh battery:**
- Sparse mode: 5-10 hours
- Dense mode: 1.5-3 hours

### 9.3 Security Professional Deployment (Path 3)

**Configuration:**
- Belt: Variant B or C
- Pendant: Variant B (360° Curved)
- Bracelets (×2): Extended with camera
- Anklets (×2): Extended
- Eyewear: Variant B or C

**Power breakdown (full active):**

| Node | Power |
|------|-------|
| Belt (SLAM + streaming) | 4000-7000 mW |
| Pendant 360° (full active) | 3500-5000 mW |
| Bracelets (×2, camera) | 500-800 mW |
| Anklets (×2) | 400-800 mW |
| Eyewear | 150-300 mW |
| **Total** | **8550-13900 mW** |

**Runtime with belt 7000 mAh + pendant belt power:**
- Full active: 30-50 minutes (pendant on belt power)
- Sparse mode: 4-8 hours

### 9.4 Long-Range Detection Deployment (Path 8)

**Configuration:**
- Belt: Variant B + piezo haptics
- Pendant: Variant F (Long-Range Detection)
- Bracelets (×2): Standard + directional haptics
- Anklets (×2): Standard
- Eyewear: Variant A (event-only)

**Power breakdown:**

| Node | Power |
|------|-------|
| Belt (sparse + piezo) | 600-1000 mW |
| Pendant (detection mode) | 300-500 mW |
| Bracelets (×2) | 100-200 mW |
| Anklets (×2) | 200-400 mW |
| Eyewear | 40-70 mW |
| **Total** | **1240-2170 mW** |

**Runtime with belt 5000 mAh:**
- Detection mode: 4-8 hours
- With pendant on belt power: unlimited

---

## 10. Operational Modes and Power States

### 10.1 System Power States

| State | Definition | Power Profile |
|-------|------------|---------------|
| **Off** | System powered down | 0 mW |
| **Deep Sleep** | All radios off, MCU in lowest power, no sensing | 5-50 mW total |
| **Sleep** | BLE advertising, sensors duty-cycled | 50-200 mW total |
| **Sparse Active** | Full BAN, sparse tracking, no SLAM | 500-1500 mW total |
| **Dense Active** | SLAM active, streaming optional | 2000-5000 mW total |
| **Full Active** | All features enabled | 4000-14000 mW total |

### 10.2 Mode Transitions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         POWER STATE TRANSITIONS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Off ──────────► Deep Sleep                                                  │
│  │            (power on)                                                     │
│  │                                                                           │
│  ▼                                                                           │
│  Deep Sleep ───► Sleep                                                       │
│  │            (user activity detected)                                       │
│  │                                                                           │
│  ▼                                                                           │
│  Sleep ────────► Sparse Active                                               │
│  │            (walk-through, calibration)                                    │
│  │                                                                           │
│  ▼                                                                           │
│  Sparse Active ─► Dense Active                                                │
│  │            (SLAM enabled, high-fidelity sensing)                          │
│  │                                                                           │
│  ▼                                                                           │
│  Dense Active ──► Full Active                                                 │
│               (streaming, all cameras, cellular active)                      │
│                                                                              │
│  Reverse transitions:                                                        │
│  - Full → Dense: streaming stopped                                          │
│  - Dense → Sparse: SLAM disabled                                            │
│  - Sparse → Sleep: no activity for configured timeout                        │
│  - Sleep → Deep Sleep: extended inactivity                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 10.3 Automatic State Transitions

**Configuration:**

```toml
[power.transitions]
# Timeouts for automatic transitions
sparse_to_sleep_minutes = 5
sleep_to_deep_sleep_minutes = 30

# Triggers for upward transitions
activity_trigger = "any_movement"
calibration_trigger = "manual"

# Mode-based power profiles
[power.profiles]
privacy_first = "sparse"
security_first = "dense"
away = "sparse"  # Streaming via cellular enabled separately
silent = "sparse"
```

### 10.4 Per-Mode Power Budgets

**Privacy-First Mode:**
- Sparse tracking only
- No SLAM, no streaming
- Identity nodes dormant
- Power: 500-1500 mW total
- Battery (5000 mAh): 7-10 hours

**Security-First Mode:**
- Dense tracking (SLAM active on belt)
- Identity nodes active on trigger
- Local recording enabled
- Power: 2000-4000 mW total
- Battery (5000 mAh): 2.5-5 hours

**Away Mode:**
- Sparse tracking
- Streaming via cellular enabled
- Power: 800-2000 mW total (depending on cellular activity)
- Battery (5000 mAh): 5-10 hours

**Silent Mode:**
- Sparse tracking
- No alerts (haptics disabled)
- Recording only
- Power: 500-1200 mW total
- Battery (5000 mAh): 8-20 hours

---

## 11. Battery Sizing and Runtime Calculations

### 11.1 Battery Chemistry and Specifications

**All SENTINEL-WEAR nodes use LiPo (Lithium Polymer) cells:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Nominal voltage | 3.7V | Discharge curve: 4.2V → 3.0V |
| Minimum voltage | 3.0V | Cutoff to protect cell |
| Maximum voltage | 4.2V | Full charge |
| Energy density | 250-300 Wh/L | Typical LiPo |
| Cycle life | 300-500 cycles | To 80% capacity |
| Self-discharge | 1-3% per month | |

### 11.2 Battery Capacity Guidelines

| Node | Recommended Capacity | Form Factor |
|------|---------------------|-------------|
| Pendant Standard | 150-300 mAh | Custom pouch or prismatic |
| Pendant 360° | 400-1000 mAh | Distributed or separate module |
| Pendant Medallion | 800-1500 mAh | Larger enclosure |
| Bracelet | 150-400 mAh | Curved pouch |
| Belt (MCU) | 2000-3000 mAh | 18650 or pouch |
| Belt (Linux SoM) | 5000-7000 mAh | 2× 18650 or large pouch |
| Belt (Hot-swap) | 5000-7000 mAh × 2 | Dual 18650 banks |
| Anklet | 300-500 mAh | Curved pouch |
| Eyewear (clip-on) | 50-80 mAh | Small pouch |
| Eyewear (frame) | 80-150 mAh | Temple segments |

### 11.3 Runtime Calculation Formula

```
Runtime (hours) = Battery Capacity (mAh) × Voltage (V) / Power Draw (mW)
Runtime (hours) = Battery Energy (mWh) / Power Draw (mW)
```

**Example: Belt Variant B in Sparse Mode**
- Battery: 5000 mAh × 3.7V = 18,500 mWh
- Power draw: 800 mW average
- Runtime: 18,500 / 800 = 23 hours

**Example: Pendant 360° in Full Active**
- Battery: 800 mAh × 3.7V = 2,960 mWh
- Power draw: 3,000 mW average
- Runtime: 2,960 / 3,000 = 0.99 hours (59 minutes)

### 11.4 Runtime Tables by Deployment

**Belt Node Variant B (Linux SoM, 5000 mAh):**

| Mode | Power | Runtime |
|------|-------|---------|
| Deep Sleep | 100 mW | 185 hours |
| Sleep | 300 mW | 62 hours |
| Sparse Active | 800 mW | 23 hours |
| Dense Active | 3,000 mW | 6 hours |
| Full Active | 5,000 mW | 3.7 hours |
| Full Active + Cellular | 6,500 mW | 2.8 hours |

**Belt Node Variant C (High-Performance, 7000 mAh):**

| Mode | Power | Runtime |
|------|-------|---------|
| Sparse Active | 1,000 mW | 26 hours |
| Dense Active + NPU | 4,500 mW | 5.8 hours |
| Full Active | 7,000 mW | 3.7 hours |
| Full Active + 5G | 9,000 mW | 2.9 hours |

**Pendant 360° (800 mAh, 8 cameras, 720p):**

| Mode | Power | Runtime | With Belt Power |
|------|-------|---------|------------------|
| Sleep | 100 mW | 30 hours | Unlimited |
| Sparse (no cameras) | 300 mW | 10 hours | Unlimited |
| Cameras Active | 3,000 mW | 1 hour | Unlimited |
| Full Streaming | 4,500 mW | 40 minutes | Unlimited |

---

## 12. Thermal Management

### 12.1 Thermal Constraints

**Jewelry form factor thermal limits:**

| Node | Max Sustained Power | Max Surface Temp | Constraint |
|------|---------------------|------------------|------------|
| Pendant Standard | ~1.5 W | 42°C | Small surface area |
| Pendant 360° | ~2.5 W | 42°C | Limited heat spreading |
| Pendant Medallion | ~2.5 W | 42°C | Larger surface helps |
| Bracelet | ~0.5 W | 42°C | Very limited |
| Belt | ~5 W | 45°C | Larger enclosure acceptable |
| Anklet | ~0.8 W | 42°C | Moderate |
| Eyewear | ~0.3 W | 40°C | Very sensitive |

### 12.2 Thermal Mitigation Strategies

**Hardware-level:**
- Thermal vias under high-power ICs
- Metal heat spreaders (copper or aluminum)
- Air gaps between PCB and enclosure
- Vented enclosures (where IP rating permits)

**Firmware-level:**
- Throttling based on temperature monitoring
- Duty cycling cameras to reduce sustained load
- SLAM frame rate reduction at high temperatures

**User-level:**
- Awareness of sustained operation limits
- Extended operation via belt power input

### 12.3 Thermal Monitoring Configuration

```toml
[thermal]
enabled = true
monitoring_interval_ms = 1000

[thermal.thresholds]
warning_c = 38              # Reduce performance
throttle_c = 42             # Significantly reduce
critical_c = 48             # Shut down high-power features
emergency_c = 55            # System shutdown

[thermal.actions]
warning = "reduce_camera_fps"
throttle = "disable_slam"
critical = "cameras_off"
emergency = "shutdown"
```

---

## 13. Deployment Path Power Analysis

### 13.1 Path 1 — Basic Awareness

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant A, 2000 mAh | MCU, sparse only |
| Pendant | Standard, no camera | Radar + IMU |
| Bracelets (×2) | Minimal | Radar + IMU |
| Anklets (×2) | Minimal | ToF + IMU |
| **Total Power** | Sparse mode | **560-1100 mW** |
| **Runtime** | Belt 2000 mAh | **4-7 hours** |

**Power per day:**
- 4 hours active: ~4 Wh
- 20 hours sleep: ~2 Wh
- Total: ~6 Wh/day

**Recommendations:**
- Nightly charging
- Suitable for casual use
- No SLAM capability

### 13.2 Path 2 — Standard Personal Awareness

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant B, 5000 mAh | Linux SoM |
| Pendant | Standard + camera | Radar + IMU + camera |
| Bracelets (×2) | Standard | Radar + IMU |
| Anklets (×2) | Extended | LiDAR + radar |
| Eyewear | Optional | |
| **Total Power (sparse)** | | **1000-1800 mW** |
| **Total Power (dense)** | | **3500-6000 mW** |
| **Runtime (sparse)** | Belt 5000 mAh | **5-10 hours** |
| **Runtime (dense)** | Belt 5000 mAh | **1.5-3 hours** |

**Power per day (mixed usage):**
- 8 hours sparse: ~12 Wh
- 2 hours dense: ~10 Wh
- 14 hours sleep: ~4 Wh
- Total: ~26 Wh/day

**Recommendations:**
- Nightly charging
- Default to sparse; dense on-demand
- SLAM available but power-intensive

### 13.3 Path 3 — Security Professional

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant B/C, 7000 mAh | Full compute |
| Pendant | 360° Curved | 8 cameras + radar |
| Bracelets (×2) | Extended + camera | Camera + radar |
| Anklets (×2) | Extended | LiDAR + radar |
| Eyewear | Variant B/C | Event + camera |
| **Total Power (full active)** | | **8500-14000 mW** |
| **Runtime (full active)** | Belt 7000 mAh | **30-50 min** |
| **With pendant on belt power** | | **~40-50 min** |

**Critical insight:** Full active mode is **short-duration only**. This is not an all-day configuration.

**Practical usage:**
- Default to sparse mode: 6-10 hours runtime
- Switch to full active for specific events: 30-60 minutes max
- Return to sparse for recovery

**Power per day (realistic professional use):**
- 6 hours sparse: ~10 Wh
- 1 hour full active: ~12 Wh
- 2 hours dense: ~6 Wh
- 15 hours sleep: ~5 Wh
- Total: ~33 Wh/day

**Recommendations:**
- Start in sparse mode
- Activate 360° pendant on event trigger
- Use belt power input for extended 360° operation
- Charging breaks during shift

### 13.4 Path 4 — Extended Runtime

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant E, 7000 mAh × 2 | Hot-swap |
| Pendant | Variant D + belt power | Tactical |
| Bracelets (×2) | Standard | |
| Anklets (×2) | Standard | |
| **Runtime** | Dual battery | **Unlimited** |

**Operational profile:**
- One battery active, one charging
- Swap every 3-8 hours depending on mode
- No downtime

**Charging logistics:**
- Carry 2-3 spare batteries
- Swap during natural breaks
- Overnight charging of all batteries

### 13.5 Path 5 — Extreme Velocity Detection

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant B, 5000 mAh | Linux SoM |
| Pendant | Variant E, CW radar + event | Detection-optimized |
| Bracelets (×2) | Standard + camera | |
| Anklets (×2) | Standard | |
| Eyewear | Variant A, event-only | Event camera |
| **Total Power (detection mode)** | | **1240-2170 mW** |
| **Runtime** | Belt 5000 mAh | **4-8 hours** |

**Detection mode power budget:**

| Component | Power |
|-----------|-------|
| Belt (sparse) | 600 mW |
| Pendant (CW radar continuous) | 150 mW |
| Pendant (event camera on standby) | 10 mW |
| Bracelets (×2) | 100 mW |
| Anklets (×2) | 200 mW |
| Eyewear (event standby) | 20 mW |
| **Total** | **1080 mW** |

**Recommendations:**
- Continuous CW radar is low-power (~150 mW)
- Detection mode can run all day
- Tier 1 reflex trigger has minimal power impact
- Piezo haptic pulses are brief (high peak, low average)

### 13.6 Path 6 — Accessibility (Visually Impaired)

| Component | Configuration | Power |
|-----------|---------------|-------|
| Belt | Variant B, 5000 mAh | Linux SoM |
| Pendant | 360° Curved | Full awareness |
| Bracelets (×2) | Standard + directional haptic | Haptic priority |
| Anklets (×2) | Standard | |
| **Total Power (sparse)** | | **1100-2000 mW** |
| **Runtime (sparse)** | Belt 5000 mAh | **5-9 hours** |

**Power considerations for accessibility:**
- Haptic alerts are frequent but brief (low average power)
- 360° pendant provides rich environmental context
- Dense mode provides full 3D world model
- Dense mode runtime limited (~1-2 hours)

**Recommendations:**
- Default to sparse mode
- Dense mode for environment learning
- Overnight charging

### 13.7 Path 7 — Research Platform

| Component | Configuration | Power |
|-----------|---------------|-------|
| All nodes | Any | Variable |
| **Power** | | **Variable** |

**Research scenarios:**
- Power measurement experiments
- Battery chemistry testing
- Thermal characterization
- Power optimization research

---

## 14. Extreme Velocity Detection Power Requirements

### 14.1 Tier 1 — Reflex Trigger

**Sensors:** CW radar (primary)

| Component | Power | Notes |
|-----------|-------|-------|
| CW radar (continuous) | 100-200 mW | Continuous Doppler monitoring |
| MCU processing | 30-80 mW | FFT + threshold |
| **Total** | **130-280 mW** | |

**This is low-power and can run continuously.**

### 14.2 Tier 2 — Direction Validation

**Sensors:** Event camera (triggered by Tier 1)

| Component | Power | Notes |
|-----------|-------|-------|
| Event camera (standby) | 10-20 mW | Low-power standby |
| Event camera (active burst) | 30-60 mW | 100-500 µs burst |
| MCU streak analysis | 50-100 mW | Line fitting |
| **Additional during Tier 2** | **90-180 mW** | |
| **Cumulative with Tier 1** | **220-460 mW** | |

### 14.3 Tier 3 — Characterization

**Sensors:** FMCW burst, conventional cameras

| Component | Power | Notes |
|-----------|-------|-------|
| FMCW radar (burst) | 200-400 mW | 500-1000 µs burst |
| Conventional camera | 100-300 mW | Recording |
| Belt processing | 500-1500 mW | Fusion, logging |
| **Additional during Tier 3** | **800-2200 mW** | |
| **Cumulative total** | **1000-2500 mW** | |

### 14.4 Continuous Detection Mode

**For users needing continuous extreme velocity detection:**

```toml
[extreme_velocity.power]
mode = "continuous_detection"
# Tier 1 always active
tier1_always_on = true
tier1_power_mw = 150

# Tier 2 on trigger
tier2_trigger = "tier1_detection"
tier2_duration_ms = 500
tier2_power_mw = 150

# Tier 3 on trigger
tier3_trigger = "tier2_confirmation"
tier3_duration_s = 2
tier3_power_mw = 1500

# Average power budget
# Idle: ~150 mW
# Event (Tier 2): ~300 mW for 500 ms
# Event (Tier 3): ~1650 mW for 2 s
# Assume 1 event per hour: negligible impact on daily power
```

---

## 15. Connectivity Power Impact

### 15.1 BLE Power

| State | Power |
|-------|-------|
| Advertising | 5-15 mW |
| Connected (idle) | 8-20 mW |
| Active transmission | 30-80 mW |

**BLE is the lowest-power connectivity option and runs on all nodes.**

### 15.2 UWB Power

| State | Power |
|-------|-------|
| Idle | 30-50 mW |
| Timing/ranging | 50-100 mW |
| Streaming | 80-150 mW |

**UWB adds ~50-150 mW when active.**

### 15.3 WiFi Power (Belt Only)

| State | Power |
|-------|-------|
| Idle | 50-150 mW |
| Connected | 100-300 mW |
| Streaming | 300-600 mW |

**WiFi adds ~300-600 mW when streaming.**

### 15.4 Cellular Power (Belt Only)

| Module | Idle | Connected | Streaming |
|--------|-------|-----------|-----------|
| LTE Cat 1 | 30 mW | 150-300 mW | 300-500 mW |
| LTE Cat 4 | 30 mW | 300-500 mW | 400-700 mW |
| 5G Sub-6 | 50 mW | 500-800 mW | 800-1200 mW |

**Cellular adds 300-1200 mW when active.**

### 15.5 Combined Connectivity Power Budgets

**Belt with all radios active:**

| Configuration | Power |
|---------------|-------|
| BLE + WiFi (idle) | 150-350 mW |
| BLE + WiFi (streaming) | 400-800 mW |
| BLE + WiFi + Cellular (streaming) | 700-2000 mW |

---

## 16. SLAM and Dense World Model Power Requirements

### 16.1 SLAM Compute Requirements

| Operation | Compute | Power |
|-----------|---------|-------|
| Feature extraction | 0.5-1 GOPS | 200-400 mW |
| Pose estimation | 0.2-0.5 GOPS | 100-200 mW |
| Loop closure | 0.1-0.2 GOPS | 50-100 mW |
| Map update | 0.1-0.3 GOPS | 50-150 mW |
| **Total (continuous SLAM)** | **1-2 GOPS** | **400-850 mW** |

### 16.2 SLAM Power by Hardware

| Platform | SLAM Performance | Power |
|----------|------------------|-------|
| MCU (STM32H7) | Not feasible | N/A |
| CM4 (Cortex-A72) | 5-10 fps | 1000-2000 mW |
| i.MX 8M Plus | 15-25 fps | 1500-2500 mW |
| Qualcomm SA8155P | 30+ fps | 2000-4000 mW |

### 16.3 Dense vs Sparse Mode Power Comparison

| Mode | Power | Features |
|------|-------|----------|
| Sparse (PentaTrack only) | 100-300 mW | Tracking, prediction |
| Dense (SLAM active) | 1500-4000 mW | Full 3D reconstruction |

**Recommendation:** Default to sparse. Activate SLAM on-demand.

---

## 17. Power Optimization Strategies

### 17.1 Duty Cycling

**Camera duty cycling:**

```toml
[camera.duty_cycle]
enabled = true
active_ms = 5000
idle_ms = 10000
# Effective camera on-time: 33%
# Power reduction: ~67%
```

**Radar duty cycling:**

```toml
[radar.duty_cycle]
enabled = true
scan_interval_ms = 100
# Reduced radar activity
# Power reduction: ~50%
```

### 17.2 Progressive Quality

**Start low, improve on demand:**

```toml
[video.progressive]
enabled = true
initial_resolution = "qvga"
target_resolution = "720p"
warmup_time_s = 10
# Power: starts at ~500 mW, rises to ~2000 mW over 10 seconds
```

### 17.3 Event-Triggered Activation

**Activate high-power features only on detection:**

```toml
[activation]
strategy = "event_triggered"

[activation.triggers]
camera = "motion_detected"
slam = "threat_detected"
streaming = "manual"

# Power: baseline low, spikes on events
```

### 17.4 Sleep Management

**Aggressive sleep for idle nodes:**

```toml
[sleep_management]
idle_timeout_s = 60
sleep_mode = "light"            # "light" | "deep"
wake_on_motion = true

# Anklets sleep when walking stops
# Bracelets sleep when no arm motion
# Pendant reduces to minimal sensing
```

### 17.5 Adaptive Performance

**Scale performance to remaining battery:**

```toml
[adaptive_performance]
enabled = true

[adaptive_performance.battery_thresholds]
# Below 50%: reduce SLAM frame rate
# Below 25%: disable SLAM
# Below 10%: sparse mode only
# Below 5%: critical sensors only

[adaptive_performance.actions]
50_percent = "slam_half_rate"
25_percent = "slam_off"
10_percent = "sparse_only"
5_percent = "minimal"
```

---

## 18. Configuration Reference

### 18.1 Power Management Configuration

```toml
# sentinel-wear.toml - Power Management Section

[power]
# Default power mode
default_mode = "sparse"         # "deep_sleep" | "sleep" | "sparse" | "dense" | "full"

# State transition timeouts
idle_timeout_minutes = 5
sleep_timeout_minutes = 30

# Adaptive performance
adaptive_enabled = true

[power.battery]
# Battery configuration
low_battery_percent = 20
critical_battery_percent = 10
reserve_battery_percent = 5

[power.thermal]
# Thermal management
monitoring_enabled = true
warning_c = 38
throttle_c = 42
critical_c = 48
emergency_c = 55

[power.profiles]
# Mode-specific power profiles
privacy_first = "sparse"
security_first = "dense"
away = "sparse"
silent = "sparse"

[power.reporting]
# Power reporting interval
interval_s = 60
```

### 18.2 Per-Node Power Configuration

```toml
[nodes.pendant]
# Pendant power configuration
camera_duty_cycle = true
camera_active_ms = 5000
camera_idle_ms = 15000
radar_duty_cycle = true
radar_scan_interval_ms = 100

[nodes.pendant_360]
# 360° pendant specific
thermal_throttle = true
max_sustained_power_mw = 2500
progressive_quality = true
initial_quality = "qvga"
target_quality = "720p"
belt_power_enabled = true

[nodes.belt]
# Belt node configuration
slam_default = "off"
slam_trigger = "manual"
streaming_trigger = "manual"
cellular_mode = "idle"

[nodes.bracelet_left]
# Bracelet power
haptic_pulse_limit_per_minute = 10

[nodes.anklet_left]
# Anklet power
gait_analysis_mode = "on_movement"
```

### 18.3 Extreme Velocity Power Configuration

```toml
[extreme_velocity.power]
# Extreme velocity detection power profile
tier1_continuous = true         # CW radar always on
tier1_power_mw = 150
tier2_on_trigger = true
tier2_power_mw = 150
tier3_on_trigger = true
tier3_power_mw = 1500
```

---

## 19. Charging Architecture

### 19.1 Charging Methods by Node

| Node | Primary Charging | Secondary |
|------|------------------|-----------|
| Pendant | USB-C or magnetic pogo | Qi wireless |
| Pendant 360° | USB-C or magnetic pogo | Belt power input |
| Bracelet | Magnetic pogo | Qi wireless |
| Belt | USB-C PD | N/A |
| Anklet | Qi wireless | Magnetic pogo |
| Eyewear (clip-on) | USB-C or pogo | N/A |
| Eyewear (frame) | Magnetic pogo | Belt cable |

### 19.2 Charging Times

**Belt Node Variant B (5000 mAh):**

| Charger | Current | Charging Time (0-100%) |
|---------|---------|------------------------|
| USB-C 5V/1A | 1A | ~5-6 hours |
| USB-C 5V/2A | 2A | ~2.5-3 hours |
| USB-C PD (9V/2A) | 2A @ 9V | ~2-2.5 hours |
| USB-C PD (15V/2A) | 2A @ 15V | ~1.5-2 hours |

**Pendant (300 mAh):**

| Charger | Current | Charging Time |
|---------|---------|---------------|
| USB-C 5V/500mA | 500mA | ~40-60 minutes |
| Qi wireless (5W) | ~1A | ~30-45 minutes |

**Bracelet (250 mAh):**

| Charger | Current | Charging Time |
|---------|---------|---------------|
| Magnetic pogo (5V/500mA) | 500mA | ~30-45 minutes |
| Qi wireless (5W) | ~800mA | ~20-30 minutes |

### 19.3 Charging Configuration

```toml
[charging]
# Charging behavior
charge_current_ma = 1000          # Default charge current
fast_charge_enabled = true
fast_charge_current_ma = 2000

# Battery protection
max_charge_voltage_mv = 4200
min_discharge_voltage_mv = 3000

# Charging from belt power (for pendant 360°)
belt_power_charging = true
belt_power_current_ma = 500      # Charge pendant from belt

# Charging indicators
led_charging_indicator = true
```

---

## 20. Monitoring and Alerts

### 20.1 Battery Monitoring

**All nodes report battery status:**

```rust
pub struct BatteryStatus {
    pub percent: u8,
    pub voltage_mv: u16,
    pub current_ma: i16,
    pub charging: bool,
    pub health: BatteryHealth,
    pub temperature_c: f32,
    pub time_remaining_minutes: Option<u32>,
}

pub enum BatteryHealth {
    Excellent,
    Good,
    Fair,
    Poor,
    Replace,
}
```

### 20.2 Power Alerts

| Alert | Threshold | Action |
|-------|-----------|--------|
| Battery Low | 20% | Warning notification |
| Battery Critical | 10% | Reduce to sparse mode |
| Battery Reserve | 5% | Minimal sensing only |
| Thermal Warning | 38°C | Reduce performance |
| Thermal Critical | 45°C | Disable high-power features |
| Charging Started | N/A | Notification |
| Charging Complete | 100% | Notification |

### 20.3 Power Monitoring Configuration

```toml
[power.monitoring]
# Monitoring interval
report_interval_s = 60

# Alert thresholds
low_battery_percent = 20
critical_battery_percent = 10
reserve_battery_percent = 5

# Thermal thresholds
warning_temp_c = 38
critical_temp_c = 45

# Notifications
alert_on_low_battery = true
alert_on_thermal = true
alert_on_charging = false
```

### 20.4 Power Analytics

**Companion app features:**

- Real-time power consumption graph
- Battery drain rate estimation
- Time-remaining prediction
- Historical power usage patterns
- Per-node power breakdown
- Charging history
- Battery health trends

```toml
[power.analytics]
# Analytics configuration
history_retention_days = 30
per_node_breakdown = true
drain_rate_calculation = true
time_remaining_prediction = true
```

---

## Appendix A: Quick Reference Tables

### A.1 Belt Node Power Summary

| Variant | Battery | Sparse Runtime | Dense Runtime | Full Active Runtime |
|---------|---------|----------------|---------------|---------------------|
| A (MCU) | 2000 mAh | 15-20 hours | N/A | N/A |
| B (Linux) | 5000 mAh | 10-15 hours | 4-6 hours | 2-3 hours |
| B (Linux) | 7000 mAh | 14-21 hours | 5-8 hours | 3-4 hours |
| C (High-perf) | 7000 mAh | 8-12 hours | 3-5 hours | 1.5-2.5 hours |
| D (ESP32) | 2000 mAh | 12-20 hours | N/A | N/A |
| E (Hot-swap) | 7000 mAh × 2 | Unlimited | Unlimited | Unlimited |

### A.2 Pendant Power Summary

| Variant | Battery | Sparse Runtime | Full Active Runtime |
|---------|---------|----------------|---------------------|
| A (Standard) | 200 mAh | 2-4 hours | 1-2 hours |
| B (360°) | 800 mAh | 3-6 hours | 0.5-1 hour |
| C (Medallion) | 1000 mAh | 5-10 hours | 2-4 hours |
| D (Tactical) | 2000 mAh | 10-20 hours | 4-8 hours |
| E (Event) | 300 mAh | 3-6 hours | 0.75-1.2 hours |
| F (Long-range) | 400 mAh | 4-8 hours | 1.5-2.5 hours |

### A.3 Bracelet Power Summary

| Config | Battery | Sparse Runtime |
|--------|---------|----------------|
| Minimal | 150 mAh | 3-6 hours |
| Standard | 250 mAh | 4-8 hours |
| Extended | 400 mAh | 6-12 hours |

### A.4 Anklet Power Summary

| Config | Battery | Active Runtime |
|--------|---------|----------------|
| Minimal | 300 mAh | 4-7.5 hours |
| Standard | 400 mAh | 3.6-6.6 hours |
| Extended | 500 mAh | 2.5-5 hours |

### A.5 Eyewear Power Summary

| Variant | Battery | Active Runtime |
|---------|---------|----------------|
| A (Event-only) | 50 mAh | 1.4-2.5 hours |
| B (Event+Camera) | 80 mAh | 1-2 hours |
| C (Full array) | 150 mAh | 1-2 hours |

---

## Appendix B: Sample Power Profile TOML

```toml
# Complete power profile configuration

[power]
default_mode = "sparse"
idle_timeout_minutes = 5
sleep_timeout_minutes = 30
adaptive_enabled = true

[power.battery]
low_battery_percent = 20
critical_battery_percent = 10
reserve_battery_percent = 5

[power.thermal]
monitoring_enabled = true
warning_c = 38
throttle_c = 42
critical_c = 48
emergency_c = 55

[power.profiles]
privacy_first = "sparse"
security_first = "dense"
away = "sparse"
silent = "sparse"

[power.transitions]
sparse_to_sleep_minutes = 5
sleep_to_deep_sleep_minutes = 30

[power.monitoring]
report_interval_s = 60
low_battery_percent = 20
critical_battery_percent = 10
warning_temp_c = 38
critical_temp_c = 45

[power.analytics]
history_retention_days = 30
per_node_breakdown = true

[charging]
fast_charge_enabled = true
led_charging_indicator = true

# Per-node overrides
[nodes.pendant_360]
thermal_throttle = true
max_sustained_power_mw = 2500
progressive_quality = true

[nodes.belt]
slam_default = "off"
slam_trigger = "manual"
streaming_trigger = "manual"

[extreme_velocity.power]
tier1_continuous = true
tier1_power_mw = 150
```

---

**End of Document**
