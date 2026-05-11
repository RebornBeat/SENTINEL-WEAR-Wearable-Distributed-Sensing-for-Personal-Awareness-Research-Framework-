# Thermal Management — Wearable Thermal Design

**Project:** SENTINEL-WEAR
**Domain:** Wearable thermal physics, heat dissipation, and user comfort
**Status:** Research Reference Design
**Implementation:** PCB thermal vias, enclosure design, firmware throttling

---

## 1. Purpose

This document specifies the thermal management architecture for SENTINEL-WEAR nodes. Wearable devices have unique thermal constraints that differ fundamentally from stationary or handheld electronics:

**The primary constraint:** Human skin has a narrow comfort band. A device at 45°C feels uncomfortably hot; at 50°C, it causes pain and potential low-temperature burns.

**The challenge:** Jewelry-form-factor devices have minimal surface area for heat dissipation, are in continuous contact with skin, and must operate for extended periods.

**The consequence:** Thermal management is not an afterthought — it is a first-class architectural concern that affects node placement, enclosure materials, firmware behavior, and user safety.

---

## 2. Thermal Physics Fundamentals

### 2.1 Heat Generation Sources in SENTINEL-WEAR Nodes

| Heat Source | Typical Power | Peak Power | Location |
|-------------|---------------|------------|----------|
| MCU (active processing) | 50-300 mW | 500 mW | All nodes |
| mmWave radar (active) | 100-500 mW | 1 W | Pendant, bracelet, belt |
| BLE radio (transmit) | 50-100 mW | 150 mW | All nodes |
| UWB radio (active) | 100-200 mW | 300 mW | Nodes with UWB |
| Camera module | 100-300 mW | 500 mW | Pendant, bracelet (camera variant) |
| Event camera | 50-150 mW | 300 mW | Eyewear, pendant (variant E) |
| Vision processor (360° pendant) | 0.5-2 W | 3 W | 360° pendant |
| Multiple cameras (8×) | 0.8-2 W | 4 W | 360° pendant |
| Linux SoM (belt, active) | 1-3 W | 5 W | Belt variant B/C |
| WiFi radio (streaming) | 300-500 mW | 800 mW | Belt only |
| Cellular module (active) | 0.5-1.5 W | 2.5 W | Belt only |
| Haptic driver (pulse) | 50-200 mW | 500 mW | Bracelet, anklet, belt |
| SD card (write) | 50-150 mW | 300 mW | Recording nodes |

### 2.2 Total Heat Budgets by Node Type

| Node | Active Mode Power | Worst-Case Heat Generation | Enclosure Surface Area |
|------|-------------------|---------------------------|------------------------|
| Pendant Standard (no camera) | 0.3-0.8 W | 1.0 W | ~25 cm² |
| Pendant Standard (with camera) | 0.5-1.2 W | 1.5 W | ~25 cm² |
| Pendant 360° (all cameras active) | 2.5-4.5 W | 6 W | ~40-60 cm² |
| Pendant Medallion | 0.5-1.5 W | 2 W | ~35 cm² |
| Bracelet (no camera) | 0.2-0.5 W | 0.8 W | ~15 cm² |
| Bracelet (with camera) | 0.4-0.8 W | 1.2 W | ~15 cm² |
| Belt MCU-only | 0.3-0.8 W | 1.5 W | ~80-150 cm² |
| Belt Linux SoM (sparse) | 1.0-2.0 W | 3 W | ~80-150 cm² |
| Belt Linux SoM (full active) | 3.0-5.0 W | 7 W | ~80-150 cm² |
| Belt with cellular streaming | 4.0-6.0 W | 9 W | ~80-150 cm² |
| Anklet | 0.2-0.5 W | 0.8 W | ~20 cm² |
| Eyewear clip-on | 0.1-0.3 W | 0.5 W | ~10 cm² |
| Eyewear frame-integrated | 0.2-0.5 W | 1.0 W | ~25 cm² |

### 2.3 Heat Flux Analysis

**Heat flux** (power per unit area) determines how hot the surface gets:

$$q = \frac{P}{A}$$

Where:
- $q$ = heat flux (W/cm²)
- $P$ = power dissipated (W)
- $A$ = effective surface area (cm²)

**Heat flux thresholds:**

| Heat Flux | Surface Temperature Rise (above ambient) | Comfort Assessment |
|-----------|------------------------------------------|-------------------|
| < 0.02 W/cm² | < 5°C | Comfortable for extended wear |
| 0.02-0.05 W/cm² | 5-10°C | Noticeable warmth, acceptable |
| 0.05-0.10 W/cm² | 10-20°C | Uncomfortable for extended contact |
| 0.10-0.20 W/cm² | 20-35°C | Hot, limited contact time |
| > 0.20 W/cm² | > 35°C | Painful, burn risk |

**Node heat flux calculations:**

| Node | Heat Flux (W/cm²) | Assessment |
|------|-------------------|------------|
| Pendant Standard (no camera) | 0.03-0.04 W/cm² | Acceptable |
| Pendant Standard (with camera, active) | 0.06 W/cm² | Noticeable warmth |
| Pendant 360° (full active) | 0.08-0.15 W/cm² | **Requires active management** |
| Bracelet (no camera) | 0.03-0.05 W/cm² | Acceptable |
| Bracelet (camera active) | 0.08 W/cm² | Noticeable warmth |
| Belt MCU-only | 0.01-0.02 W/cm² | Excellent |
| Belt Linux SoM (full active) | 0.04-0.09 W/cm² | Requires management |
| Belt with cellular | 0.06-0.11 W/cm² | **Requires active management** |
| Anklet | 0.02-0.04 W/cm² | Acceptable |
| Eyewear clip-on | 0.02-0.05 W/cm² | Acceptable |

**Key insight:** The 360° pendant and the belt node under full load require active thermal management. Other nodes are thermally acceptable with passive design.

---

## 3. Skin Temperature Limits

### 3.1 Physiological Basis

Human skin temperature is normally 32-34°C. The comfort and safety limits are:

| Temperature | Sensation | Maximum Contact Time |
|-------------|-----------|---------------------|
| < 35°C | Neutral/cool | Unlimited |
| 35-38°C | Slightly warm | Unlimited |
| 38-40°C | Warm | Several hours |
| 40-42°C | Noticeably warm | 1-2 hours before discomfort |
| 42-44°C | Hot | 30-60 minutes |
| 44-46°C | Very hot | 5-15 minutes |
| 46-48°C | Pain threshold | 1-5 minutes |
| > 48°C | Burn risk | Seconds to minutes |

### 3.2 Design Targets

**For extended wear (8+ hours continuous):**
- Maximum enclosure surface temperature: **40°C**
- Ambient temperature assumption: 25°C (typical indoor)
- Maximum rise above ambient: 15°C

**For intermittent high-power operation (streaming, SLAM):**
- Maximum enclosure surface temperature: **45°C** (limited duration)
- Time limit: 30 minutes at 45°C before firmware throttling required
- Cool-down period required after extended high-power operation

**For belt node (not in direct skin contact on all surfaces):**
- Interior (skin contact): **42°C maximum**
- Exterior (non-contact): **50°C acceptable** (not burning to touch)

---

## 4. Thermal Design by Node Type

### 4.1 Pendant Standard (Variants A, C)

**Thermal challenge:** Moderate heat generation, small surface area, skin contact on rear face

**Heat sources:**
- MCU: 50-150 mW
- mmWave radar: 100-300 mW
- Acoustic processing: 50-100 mW
- BLE radio: 50 mW average
- Optional camera: 100-300 mW

**Total continuous:** 250-600 mW (no camera), 350-900 mW (with camera)

**Thermal design:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PENDANT STANDARD THERMAL                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Front Face (exterior)                │    │
│  │                                                         │    │
│  │   [Sensor Windows]   [Air Gap]   [Sensor Windows]      │    │
│  │                                                         │    │
│  │   Sensor windows are RF/acoustic transparent;           │    │
│  │   remainder is solid metal/plastic for heat spreading   │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            ▲                                    │
│                            │ Heat flow                          │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                       PCB                               │    │
│  │                                                         │    │
│  │   [MCU] ────┐                                          │    │
│  │             │ Thermal vias                              │    │
│  │             ▼                                          │    │
│  │   [Radar] ───────► Thermal pad ────► Metal back plate   │    │
│  │                                                         │    │
│  │   Thermal vias under high-power ICs (0.3 mm pitch,     │    │
│  │   0.2 mm diameter, 20-40 vias per IC)                   │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Rear Face (skin contact)                │    │
│  │                                                         │    │
│  │   Thin medical-grade silicone between metal back        │    │
│  │   plate and skin (provides comfort, spreads heat)       │    │
│  │                                                         │    │
│  │   Target: 40°C maximum during normal operation          │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Design elements:**

| Element | Specification | Purpose |
|---------|---------------|---------|
| Thermal vias | 0.3 mm pitch, 0.2 mm diameter | Conduct heat from IC to back plate |
| Metal back plate | 0.5-1.0 mm titanium or aluminum | Spread heat across rear surface |
| Silicone layer | 0.5-1.0 mm medical silicone | Comfort layer, moderate thermal resistance |
| Air gap (front) | 0.5-1.0 mm | Insulate front sensors from heat |
| Enclosure material | Metal (primary), plastic (sensor windows) | Metal spreads heat; plastic insulates sensors |

### 4.2 Pendant 360° Curved (Variant B)

**Thermal challenge:** High heat generation (2.5-4.5 W), curved form factor, extended runtime scenarios

**Heat sources:**
- Vision processor: 0.5-2 W
- 8× cameras: 0.8-2 W
- mmWave radar arrays: 200-500 mW
- IMU arrays: 50-100 mW
- BLE/UWB: 100-200 mW
- Stitching compute: 0.5-1 W

**Total continuous (full active):** 2.5-4.5 W
**Peak (during burst):** 6 W

**Thermal design:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    360° PENDANT THERMAL ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  [Camera 0]─[Camera 1]─[Camera 2]─[Camera 3]─[Camera 4]─[Camera 5]─...   │
│       │          │          │          │          │          │            │
│       │          │          │          │          │          │            │
│       ▼          ▼          ▼          ▼          ▼          ▼            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Flexible PCB Arc                                │  │
│  │                                                                   │  │
│  │   Camera zones: High-density power distribution, local bypass    │  │
│  │   cap per camera                                                  │  │
│  │                                                                   │  │
│  │   Thermal relief: Flex PCB has limited thermal conductivity;      │  │
│  │   heat remains localized near vision processor                    │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│                              │ Central zone                              │
│                              ▼                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Central Vision Processor                       │  │
│  │                                                                   │  │
│  │   [Vision MCU/ISP] — Primary heat source (1-2 W)                 │  │
│  │          │                                                        │  │
│  │          ▼                                                        │  │
│  │   Large thermal pad (10×10 mm) to enclosure                      │  │
│  │          │                                                        │  │
│  │          ▼                                                        │  │
│  │   Thermal vias (50+ vias, 0.4 mm pitch) to outer shell          │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│                              ▼                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Enclosure Shell                                │  │
│  │                                                                   │  │
│  │   Material: Titanium (preferred) or aluminum                      │  │
│  │   Wall thickness: 1.0-1.5 mm                                     │  │
│  │   Surface area: 40-60 cm² (depends on arc length)                │  │
│  │                                                                   │  │
│  │   Heat spreading: Entire arc acts as heat spreader               │  │
│  │                                                                   │  │
│  │   External surface: Exposed to air (natural convection)          │  │
│  │   Internal surface: Limited skin contact (front only)            │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| Vision processor at center | Centralized heat generation, easier to manage |
| Metal arc as heat spreader | Entire arc distributes heat; curved shape has more surface area |
| Duty cycling cameras | Reduce average power when not actively streaming |
| Active cooling vents | Required for extended runtime operation |
| Belt power input | For extended runtime, draw power from belt (no battery heat) |

**Duty cycling strategy:**

```toml
[pendant_360.thermal]
# Active thermal management for extended operation
mode = "adaptive"                 # "passive" | "active" | "adaptive"

[pendant_360.thermal.adaptive]
# Reduce camera frame rate when thermal limit approached
hot_threshold_c = 38              # Reduce load above 38°C
critical_threshold_c = 42         # Aggressive throttling above 42°C
shutdown_threshold_c = 45         # Disable cameras above 45°C

# Duty cycling
camera_duty_cycle_at_hot = 0.7    # 70% frame rate when hot
camera_duty_cycle_at_critical = 0.3  # 30% frame rate when critical
```

### 4.3 Belt Node (All Variants)

**Thermal challenge:** Highest heat generation, but larger surface area, not all surfaces in skin contact

**Heat sources by variant:**

| Variant | Continuous Power | Peak Power | Primary Heat Sources |
|---------|------------------|------------|---------------------|
| A — MCU | 0.3-0.8 W | 1.5 W | MCU, BLE, mmWave |
| B — Linux SoM (sparse) | 1.0-2.0 W | 3 W | SoM, WiFi, BLE |
| B — Linux SoM (full active) | 3.0-5.0 W | 7 W | SoM, WiFi, SLAM, stitching |
| B — with cellular | 4.0-6.0 W | 9 W | All above + cellular |
| C — High-performance | 4.0-7.0 W | 10 W | GPU, NPU, WiFi, cellular |
| D — ESP32 | 0.5-1.0 W | 2 W | ESP32, WiFi, BLE |

**Thermal design:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       BELT NODE THERMAL ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Exterior Face (away from body)                 │  │
│  │                                                                   │  │
│  │   [Ventilation Slots] ──► Air flow for active cooling            │  │
│  │                                                                   │  │
│  │   Material: Metal (aluminum or titanium) for heat spreading      │  │
│  │                                                                   │  │
│  │   Target: 50°C maximum (exterior is not in continuous contact)   │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                              ▲                                           │
│                              │ Heat flow                                 │
│                              │                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                       PCB Layer                                   │  │
│  │                                                                   │  │
│  │   [SoM/CPU]──┐                                                   │  │
│  │              │ Thermal vias (large array)                        │  │
│  │              ▼                                                   │  │
│  │   [PMIC]─────┼──► Heat spreader (copper plane)                   │  │
│  │              │                                                   │  │
│  │   [Cellular]─┘                                                   │  │
│  │                                                                   │  │
│  │   Thermal vias: 100+ vias under SoM, 50+ under cellular module   │  │
│  │   Ground plane: 2 oz copper on all layers (thermal mass)         │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│                              ▼                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    Interior Face (body contact)                   │  │
│  │                                                                   │  │
│  │   Medical-grade silicone layer (1-2 mm)                          │  │
│  │                                                                   │  │
│  │   This layer:                                                    │  │
│  │   - Provides comfort against skin                               │  │
│  │   - Provides moderate thermal resistance                         │  │
│  │   - Reduces peak temperature felt by wearer                     │  │
│  │                                                                   │  │
│  │   Target: 42°C maximum at silicone surface                      │  │
│  │                                                                   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Active cooling (Variant B/C under full load):**

For Linux SoM variants under full load (3-7 W), passive cooling alone cannot maintain acceptable skin temperatures. Active cooling is required:

| Active Cooling Method | Implementation | Effectiveness |
|-----------------------|-----------------|----------------|
| Ventilation slots | Cutouts in enclosure, natural convection | Moderate |
| Internal heat spreader | Copper or aluminum plate | Good |
| Forced air (fan) | Miniature blower, rare in wearables | Excellent (if implemented) |
| Phase change material | PCM pouch inside enclosure | Limited duration cooling |

**Recommended approach for belt node:**

```toml
[belt.thermal]
# Thermal management configuration
mode = "hybrid"                   # "passive" | "active" | "hybrid"

[belt.thermal.passive]
# Passive design elements
heat_spreader_thickness_mm = 1.5
ventilation_slots = true
slot_count = 8
slot_width_mm = 3
slot_length_mm = 15

[belt.thermal.active]
# Firmware-based thermal management
monitoring_enabled = true
thermistor_count = 2              # Interior and exterior
throttling_enabled = true

[belt.thermal.thresholds]
warning_c = 40                    # Log warning
throttle_c = 45                   # Reduce SLAM frame rate, camera FPS
critical_c = 50                   # Disable non-essential functions
shutdown_c = 55                   # Emergency shutdown
```

### 4.4 Bracelet Node

**Thermal challenge:** Small surface area, continuous skin contact, moderate heat generation

**Heat sources:**
- MCU: 30-100 mW
- mmWave radar: 100-300 mW
- BLE: 30-50 mW
- Haptic: 50-200 mW (intermittent)
- Optional camera: 100-300 mW

**Total continuous (no camera):** 160-450 mW
**Total continuous (with camera):** 260-750 mW

**Heat flux:** 0.03-0.08 W/cm²

**Thermal design:**

| Element | Specification |
|---------|---------------|
| Enclosure material | Metal (titanium/steel) for heat spreading |
| Rear face (skin contact) | Medical silicone or thin metal with silicone coating |
| Thermal vias | 20-30 vias under MCU and radar |
| Design margin | Comfortable for extended wear without camera; warm with camera active |

**The bracelet's small form factor means heat spreading is limited.** For camera-equipped variants, firmware duty cycles the camera to reduce average heat generation.

### 4.5 Anklet Node

**Thermal challenge:** Small surface area, intermittent skin contact (during motion), moderate heat generation

**Heat sources:**
- MCU: 30-100 mW
- ToF/LiDAR: 50-150 mW
- IMU: 20-50 mW
- BLE: 30-50 mW
- Haptic: 50-150 mW (intermittent)

**Total continuous:** 130-350 mW

**Heat flux:** 0.02-0.04 W/cm²

**Assessment:** Thermally acceptable for extended wear. Natural leg motion creates air circulation around the anklet, improving heat dissipation.

### 4.6 Eyewear Node

**Thermal challenge:** Very small surface area, but on ears/face (highly sensitive to heat)

**Heat sources:**
- MCU: 20-80 mW
- Event camera: 50-150 mW
- IMU: 20-30 mW
- BLE: 20-40 mW

**Total continuous:** 110-300 mW

**Heat flux:** 0.02-0.05 W/cm² (clip-on), 0.01-0.02 W/cm² (frame-integrated)

**Critical constraint:** The head and ears are highly thermally sensitive. A warm eyewear device is immediately noticeable and uncomfortable.

**Design approach:**

| Design Element | Specification |
|----------------|---------------|
| Heat-generating components | Positioned away from skin-contact areas |
| Thermal insulation | Silicone or plastic barrier between PCB and skin |
| Active duty cycling | Event camera enabled only when needed |
| Power budget | Limited to 300 mW maximum continuous |

---

## 5. Material Selection for Thermal Management

### 5.1 Enclosure Materials

| Material | Thermal Conductivity | Density | Use Case |
|----------|---------------------|---------|----------|
| Grade 5 Titanium (Ti-6Al-4V) | 6.7 W/m·K | 4.43 g/cm³ | Premium enclosures, excellent heat spreading |
| 316L Stainless Steel | 16 W/m·K | 7.99 g/cm³ | Standard enclosures, good heat spreading |
| Aluminum 6061 | 167 W/m·K | 2.70 g/cm³ | Best thermal conductivity, requires coating for skin contact |
| Polycarbonate | 0.2 W/m·K | 1.20 g/cm³ | Insulator, used for sensor windows |
| PETG | 0.2 W/m·K | 1.27 g/cm³ | 3D printed prototypes, insulator |

### 5.2 Thermal Interface Materials

| Material | Thermal Conductivity | Thickness | Use Case |
|----------|---------------------|-----------|----------|
| Thermal pad (silicone-based) | 1-6 W/m·K | 0.5-2.0 mm | Between IC and enclosure |
| Thermal paste | 4-8 W/m·K | 0.1-0.3 mm | High-performance contact |
| Graphite sheet | 10-20 W/m·K | 0.1-0.5 mm | In-plane heat spreading |
| Thermal tape | 0.5-2 W/m·K | 0.1-0.3 mm | Low-cost assembly |

### 5.3 Skin-Contact Materials

| Material | Thermal Conductivity | Comfort | Notes |
|----------|---------------------|---------|-------|
| Medical-grade silicone | 0.2 W/m·K | Excellent | Standard for skin contact |
| Silicone + ceramic fill | 0.5-1.0 W/m·K | Good | Improved heat transfer |
| Titanium (bare) | 6.7 W/m·K | Acceptable | Feels cold initially, then warm |
| Titanium + PVD coating | 6.7 W/m·K | Good | Coating provides comfort |

---

## 6. PCB Thermal Design

### 6.1 Thermal Vias

**Purpose:** Conduct heat from IC packages through PCB to heat spreader or enclosure

**Standard thermal via specification:**

| Parameter | Value |
|-----------|-------|
| Via diameter | 0.2-0.3 mm |
| Via pitch | 0.3-0.5 mm |
| Plating | Copper, 25-35 µm |
| Fill | Copper-filled (optional, for high-power ICs) |
| Count under typical IC | 20-40 vias |
| Count under high-power IC | 50-100 vias |

**Placement:**
- Under SoC/MCU (especially Linux SoM)
- Under mmWave radar modules
- Under vision processors
- Under PMIC/power converters

### 6.2 Copper Planes

**Purpose:** Spread heat across PCB surface area, increase thermal mass

**Specification:**

| Layer | Copper Weight | Purpose |
|-------|---------------|---------|
| Top signal | 1 oz (35 µm) | Component mounting, moderate spreading |
| Ground plane | 2 oz (70 µm) | Thermal mass, primary spreading |
| Power plane | 2 oz (70 µm) | Thermal mass, secondary spreading |
| Bottom signal | 1 oz (35 µm) | Component mounting |

**High-power variants:** Consider 4-layer or 6-layer PCB with multiple 2 oz copper planes.

### 6.3 Thermal Relief Patterns

For components requiring thermal management:

```
┌──────────────────────────────────────────────┐
│           Thermal Via Array Pattern           │
│                                              │
│    •   •   •   •   •   •   •   •            │
│                                              │
│    •   •   •   •   •   •   •   •            │
│                                              │
│    •   •   •   •   •   •   •   •            │
│           (each • = thermal via)             │
│                                              │
│    Via pitch: 0.4 mm                         │
│    Via diameter: 0.25 mm                     │
│    Array size: matched to IC package         │
│                                              │
└──────────────────────────────────────────────┘
```

---

## 7. Firmware Thermal Management

### 7.1 Thermal Monitoring

**Hardware:**
- NTC thermistor(s) on PCB
- Placement: near heat-generating ICs
- Connection: ADC input to MCU

**Monitoring configuration:**

```toml
[thermal.monitoring]
enabled = true
sensor_count = 2                  # Interior and exterior
read_interval_ms = 1000
averaging_samples = 4
```

### 7.2 Thermal Thresholds and Actions

| Threshold | Temperature | Action |
|-----------|-------------|--------|
| Normal | < 38°C | All features enabled |
| Warm | 38-42°C | Log warning, prepare for throttling |
| Hot | 42-45°C | Throttle: reduce SLAM frame rate, camera FPS |
| Critical | 45-50°C | Disable: SLAM, camera streaming, cellular |
| Emergency | > 50°C | Shutdown all non-essential functions |

**Firmware implementation:**

```rust
/// Thermal management state machine
pub enum ThermalState {
    Normal,      // < 38°C — all features enabled
    Warm,        // 38-42°C — prepare for throttling
    Hot,         // 42-45°C — throttle high-power features
    Critical,    // 45-50°C — disable non-essential functions
    Emergency,   // > 50°C — shutdown
}

impl ThermalManager {
    pub fn update(&mut self) {
        let temp = self.read_temperature();
        
        self.state = match temp {
            t if t < 38.0 => ThermalState::Normal,
            t if t < 42.0 => ThermalState::Warm,
            t if t < 45.0 => ThermalState::Hot,
            t if t < 50.0 => ThermalState::Critical,
            _ => ThermalState::Emergency,
        };
        
        self.apply_state_actions();
    }
    
    fn apply_state_actions(&mut self) {
        match self.state {
            ThermalState::Normal => {
                self.slam_frame_rate_hz = 10;
                self.camera_fps = 30;
                self.streaming_enabled = true;
            }
            ThermalState::Warm => {
                // Prepare for throttling, no action yet
                self.log_warning();
            }
            ThermalState::Hot => {
                // Throttle
                self.slam_frame_rate_hz = 5;      // Reduce by half
                self.camera_fps = 15;             // Reduce by half
                self.streaming_bitrate_kbps = 2000; // Lower quality
            }
            ThermalState::Critical => {
                // Disable high-power features
                self.slam_enabled = false;
                self.camera_fps = 5;              // Minimal frame rate
                self.streaming_enabled = false;
                self.cellular_enabled = false;
            }
            ThermalState::Emergency => {
                // Shutdown non-essential
                self.slam_enabled = false;
                self.camera_enabled = false;
                self.streaming_enabled = false;
                self.cellular_enabled = false;
                // Keep only BLE and basic sensing
            }
        }
    }
}
```

### 7.3 Duty Cycling

**Purpose:** Reduce average power by operating high-power features intermittently

**Strategies:**

| Feature | Normal | Duty-Cycled (Hot) | Duty-Cycled (Critical) |
|---------|--------|-------------------|------------------------|
| SLAM frame rate | 10 Hz | 5 Hz | Off |
| Camera frame rate | 30 fps | 15 fps | 5 fps |
| Streaming bitrate | 4000 kbps | 2000 kbps | Off |
| Cellular streaming | Enabled | Disabled | Disabled |

**Configuration:**

```toml
[thermal.throttling]
# Frame rate reduction
slam_rate_normal_hz = 10
slam_rate_hot_hz = 5
slam_rate_critical_hz = 0

camera_fps_normal = 30
camera_fps_hot = 15
camera_fps_critical = 5

# Streaming quality
stream_bitrate_normal_kbps = 4000
stream_bitrate_hot_kbps = 2000
stream_bitrate_critical_kbps = 0

# Cellular
cellular_normal = true
cellular_hot = false
cellular_critical = false
```

---

## 8. Thermal Testing and Validation

### 8.1 Test Setup

**Equipment:**
- Thermal camera (FLIR or equivalent)
- Thermocouples (type K, 0.1°C accuracy)
- Environmental chamber (optional)
- Power supply and measurement
- Test fixture to simulate skin contact

### 8.2 Test Procedures

**Test 1: Steady-State Temperature**

| Step | Action | Duration |
|------|--------|----------|
| 1 | Mount thermocouples at skin-contact locations | — |
| 2 | Power on node, enable all features | — |
| 3 | Run at maximum continuous power | Until thermal equilibrium |
| 4 | Record temperatures at skin-contact surfaces | 5 minutes at equilibrium |
| 5 | Verify: skin-contact surface < 42°C | PASS/FAIL |

**Acceptance criteria:**

| Node | Max Skin-Contact Temp | Max Exterior Temp | Duration |
|------|----------------------|-------------------|----------|
| Pendant Standard (no camera) | 40°C | 45°C | 4 hours |
| Pendant Standard (camera active) | 42°C | 48°C | 2 hours |
| Pendant 360° (full active) | 45°C | 52°C | 1 hour |
| Bracelet (no camera) | 40°C | 45°C | 8 hours |
| Bracelet (camera active) | 43°C | 48°C | 4 hours |
| Belt MCU-only | 40°C | 50°C | 12 hours |
| Belt Linux SoM (sparse) | 42°C | 50°C | 8 hours |
| Belt Linux SoM (full active) | 45°C | 52°C | 2 hours |
| Anklet | 40°C | 45°C | 8 hours |
| Eyewear | 38°C | 42°C | 4 hours |

**Test 2: Transient Response**

| Step | Action | Measurement |
|------|--------|-------------|
| 1 | Start from cold (ambient) | t = 0 |
| 2 | Enable all high-power features | — |
| 3 | Record temperature vs. time | Every 10 seconds for 10 minutes |
| 4 | Measure time to reach warning threshold | Target: > 2 minutes |
| 5 | Measure time to reach critical threshold | Target: > 5 minutes |

**Acceptance criteria:**
- Time to 42°C skin-contact temperature: **> 2 minutes** (gives user time to notice and react)
- Time to 45°C skin-contact temperature: **> 5 minutes** (gives firmware time to throttle)

**Test 3: Thermal Throttling Verification**

| Step | Action | Verification |
|------|--------|---------------|
| 1 | Configure thermal throttling thresholds | — |
| 2 | Run node at full power until hot threshold | Temperature reaches 42°C |
| 3 | Verify firmware throttles features | SLAM rate, camera FPS reduced |
| 4 | Verify temperature stabilizes or decreases | Post-throttle temp < 42°C |
| 5 | Continue until critical threshold | Temperature reaches 45°C |
| 6 | Verify firmware disables high-power features | SLAM, streaming disabled |
| 7 | Verify temperature decreases | Post-critical temp < 45°C |

**Test 4: Environmental Chamber**

| Condition | Temperature | Duration | Verification |
|-----------|-------------|----------|---------------|
| Cold | 0°C | 2 hours | Skin-contact temp < 40°C (device heats itself) |
| Room | 25°C | 4 hours | Per steady-state criteria |
| Hot | 35°C | 2 hours | Skin-contact temp < 45°C |
| Hot + humid | 35°C / 80% RH | 2 hours | No condensation, temp criteria |

---

## 9. Thermal Design Checklist

### 9.1 PCB Design

- [ ] Thermal vias under all high-power ICs (MCU, radar, vision processor)
- [ ] Copper planes on multiple layers for heat spreading
- [ ] NTC thermistor placed near primary heat source
- [ ] Power distribution sized for maximum current with margin
- [ ] Ground plane connected to thermal vias for heat spreading

### 9.2 Enclosure Design

- [ ] Metal surfaces for heat spreading (not all plastic)
- [ ] Thermal interface material between PCB and enclosure
- [ ] Silicone layer or other barrier for skin comfort
- [ ] Ventilation slots for air circulation (where design allows)
- [ ] No sharp edges that concentrate heat

### 9.3 Firmware

- [ ] Thermal monitoring task reads temperature periodically
- [ ] Thermal state machine implements all states (Normal → Emergency)
- [ ] Throttling actions reduce heat generation
- [ ] Emergency shutdown when thresholds exceeded
- [ ] Thermal events logged for analysis

### 9.4 Testing

- [ ] Steady-state temperature test passed
- [ ] Transient response test passed
- [ ] Thermal throttling verification test passed
- [ ] Environmental chamber test passed (if applicable)

---

## 10. Configuration Reference

### 10.1 Complete Thermal Configuration

```toml
[thermal]
# Master enable
monitoring_enabled = true

# Thermistor configuration
thermistor_count = 2
thermistor_type = "NTC_10K"
thermistor_beta = 3950
adc_resolution_bits = 12
read_interval_ms = 1000
averaging_samples = 4

# Temperature thresholds (°C)
[thermal.thresholds]
normal_max = 38
warm_start = 38
hot_start = 42
critical_start = 45
emergency_start = 50

# State machine behavior
[thermal.states.normal]
all_features_enabled = true

[thermal.states.warm]
log_level = "warning"
prepare_throttling = true

[thermal.states.hot]
log_level = "warning"
throttle_slam = true
slam_rate_multiplier = 0.5
throttle_camera = true
camera_fps_multiplier = 0.5
throttle_streaming = true
streaming_bitrate_multiplier = 0.5

[thermal.states.critical]
log_level = "error"
disable_slam = true
disable_streaming = true
disable_cellular = true
minimal_camera_fps = 5

[thermal.states.emergency]
log_level = "error"
disable_all_nonessential = true
keep_ble_only = true

# Duty cycling
[thermal.duty_cycling]
enabled = true
hot_state_cycle_period_ms = 5000    # Re-evaluate every 5 seconds
cooldown_period_s = 60               # Require cool-down before re-enabling

# Safety
[thermal.safety]
emergency_shutdown = true
shutdown_threshold_c = 55
hysteresis_c = 3                    # Must cool 3°C below threshold to exit state
```

---

## 11. Summary

### 11.1 Key Design Principles

1. **Heat flux is the critical metric** — W/cm² determines surface temperature
2. **360° pendant and belt node require active thermal management** — others acceptable with passive design
3. **Firmware throttling is mandatory** — hardware alone cannot solve thermal challenge
4. **Skin temperature limits are non-negotiable** — 42°C maximum for extended wear
5. **Duty cycling reduces average power** — essential for camera-heavy variants

### 11.2 Node-Specific Guidance

| Node | Thermal Strategy |
|------|------------------|
| Pendant Standard | Passive: metal back plate, thermal vias, silicone layer |
| Pendant 360° | Active: duty cycling, thermal throttling, belt power option |
| Bracelet | Passive with duty cycling for camera variant |
| Belt MCU | Passive: adequate surface area |
| Belt Linux SoM | Active: thermal throttling, ventilation, firmware management |
| Anklet | Passive: motion provides air circulation |
| Eyewear | Passive with strict power limits |

### 11.3 Implementation Priority

1. **Firmware thermal management** — Immediate safety and feature control
2. **PCB thermal vias and copper planes** — Foundation for heat dissipation
3. **Metal enclosure with thermal interface** — Heat spreading to environment
4. **Silicone comfort layer** — Skin safety and comfort
5. **Ventilation and active cooling** — For highest-power variants

---

**End of Document**
