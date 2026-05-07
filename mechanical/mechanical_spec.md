# Mechanical Specification — SENTINEL-WEAR Enclosures

**Version:** 0.2 | **Status:** Research Reference Design

---

## 1. Overview

SENTINEL-WEAR enclosures must satisfy jewelry-grade aesthetic requirements while housing sensing PCBs, batteries, antennas, and optional camera modules. The design philosophy: **components follow the jewelry form, not the other way around.**

This document specifies mechanical enclosures for all SENTINEL-WEAR node types, including thermal management, weight distribution, sensor apertures, water resistance, hypoallergenic requirements, and human factors considerations.

---

## 2. Design Principles

### 2.1 Jewelry-First Philosophy

- Enclosure forms are derived from traditional jewelry archetypes (pendants, bracelets, anklets, belt buckles)
- Technology is embedded within the jewelry form, not displayed prominently
- Visual aesthetics prioritize elegance over overt technology appearance
- Users should feel comfortable wearing the system in social, professional, and casual contexts

### 2.2 Functional Transparency

- Sensor apertures are integrated aesthetically, not visually prominent
- Antennas and windows blend with the jewelry design
- Haptic feedback is felt, not seen
- LEDs and status indicators are subtle or hidden when not active

### 2.3 Wearability First

- Weight distribution prevents fatigue
- Contact surfaces are hypoallergenic
- Thermal management prevents discomfort
- Form factor accommodates diverse body types and activities

---

## 3. Material Standards

### 3.1 Skin-Contact Materials (EN 1811:2011 Compliant)

| Material | Application | Properties |
|----------|-------------|------------|
| Grade 5 Titanium (Ti-6Al-4V) | Enclosure shells, clips, buckles | Lightweight, hypoallergenic, excellent strength-to-weight |
| 316L Surgical Stainless Steel | Enclosure shells, bands | Hypoallergenic, corrosion resistant, affordable |
| Medical-Grade Silicone (ISO 10993) | Bands, straps, contact surfaces | Soft, flexible, biocompatible |
| Hard-Coat Anodized Aluminum | Enclosure shells | Lightweight, good thermal conductivity, coating prevents skin contact |
| PTFE (Teflon) | Acoustic membranes, gaskets | Biocompatible, water resistant |
| Polycarbonate | Optical windows | Biocompatible grades available, optical clarity |

### 3.2 Non-Skin-Contact Materials

| Material | Application | Properties |
|----------|-------------|------------|
| ABS Plastic | Internal structures, non-contact surfaces | Low cost, easy to mold |
| PETG | Internal structures, rapid prototype enclosures | Easy 3D printing, moderate strength |
| Polyimide (Kapton) | Flexible PCB substrate | High temperature tolerance, flexible |
| FR-4 | Rigid PCB substrate | Standard PCB material |
| Copper | PCB traces, shields | Electrical conductivity |
| Nickel-plated components | Internal hardware | Avoid direct skin contact |

### 3.3 Prohibited Materials for Skin Contact

- 6061/7075 aluminum alloys (uncoated)
- Standard (non-surgical) stainless steel
- Brass
- Unplated zinc die-cast
- Nickel-plated surfaces without barrier coating
- Chromium-plated surfaces without underplating

---

## 4. Pendant Node Enclosures

### 4.1 Variant A — Standard Flat Pendant

**Target form:** Medallion, flat pendant, or lozenge shape.

**Dimensions:**
- Maximum: 50 mm × 50 mm × 15 mm
- Target weight: 30-60 g (with battery)
- Battery: 150-300 mAh

**Enclosure construction:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Standard Pendant Enclosure                   │
├─────────────────────────────────────────────────────────────────┤
│  Front Face (exterior)                                          │
│  - Material: Grade 5 titanium shell with polycarbonate window   │
│  - Window: RF-transparent polycarbonate for mmWave radar       │
│  - Finish: Brushed or polished titanium                         │
├─────────────────────────────────────────────────────────────────┤
│  Side Walls                                                     │
│  - Material: Titanium shell                                     │
│  - Features: Microphone ports (4 positions, PTFE membrane)     │
│  - Environmental vents: Bottom edge, water-resistant design    │
├─────────────────────────────────────────────────────────────────┤
│  Back Face (body-contact)                                       │
│  - Material: Medical-grade silicone pad or titanium            │
│  - Finish: Matte for skin comfort                              │
│  - Features: Charging contacts (POGO pins) or USB-C port       │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - PCB: 45 mm × 45 mm maximum                                   │
│  - Battery: Flat LiPo pouch, 150-300 mAh                       │
│  - Sensors: mmWave radar, IMU, microphone array                │
│  - Optional camera: Forward-facing through front window        │
└─────────────────────────────────────────────────────────────────┘
```

**Sensor apertures:**

| Sensor | Aperture Type | Material | Position |
|--------|---------------|----------|----------|
| mmWave radar | RF-transparent window | Polycarbonate, no metal traces | Front face, centered |
| IMU | None required | Internal | N/A |
| Microphone array | Acoustic ports | PTFE membrane waterproof | 4 ports on sides |
| Camera (optional) | Optical window | Clear polycarbonate, AR coating | Front face |
| Environmental | Ventilation slots | Gauze filter | Bottom edge |

**Attachment:**
- Standard necklace bail (top center)
- Compatible with chains up to 3 mm diameter
- Alternative: Integrated necklace loop (for dedicated chain)

**IP Rating:** IP67 (1 m submersion, 30 minutes)

**Charging options:**
- Magnetic POGO pins (rear face)
- USB-C port (side or bottom edge)
- Qi wireless charging (if no metal interference with NFC)

**Thermal considerations:**
- Maximum power dissipation: ~1 W sustained
- Thermal vias under MCU
- Air gap between PCB and enclosure
- Maximum skin-contact temperature: 42°C

### 4.2 Variant B — 360° Curved Pendant

**Target form:** Arc-shaped pendant following the necklace curve.

**Dimensions:**
- Arc length: 120-160 mm
- Width: 15 mm
- Thickness: 10-15 mm (at thickest point)
- Target weight: 50-120 g (depending on camera count and battery)
- Battery: 400-800 mAh (distributed or separate module)

**Enclosure construction:**

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    360° Curved Pendant Enclosure                          │
├───────────────────────────────────────────────────────────────────────────┤
│  Outer Arc (exterior, away from body)                                    │
│  - Material: Curved titanium or surgical steel channel                   │
│  - Camera windows: One optical window per camera position                │
│  - Window material: AR-coated polycarbonate                              │
│  - Finish: Brushed or polished metal                                     │
├───────────────────────────────────────────────────────────────────────────┤
│  Inner Arc (body-contact)                                                │
│  - Material: Medical-grade silicone pad                                  │
│  - Purpose: Comfort and thermal isolation                                │
│  - Features: May include conductive traces for belt power input          │
├───────────────────────────────────────────────────────────────────────────┤
│  End Caps                                                                │
│  - Material: Titanium or stainless steel                                 │
│  - Features: Chain attachment points, antenna clearance                  │
│  - May house additional battery capacity                                 │
├───────────────────────────────────────────────────────────────────────────┤
│  Internal Architecture                                                   │
│  - PCB: Curved rigid-flex following arc shape                           │
│  - Cameras: 4-8 modules distributed around circumference                 │
│  - Vision processor: Central position or near highest thermal mass       │
│  - mmWave radar arrays: Multiple positions for 360° coverage            │
│  - Battery: Distributed along arc or in end caps                        │
│  - Thermal: Significant (2-4 W active)                                   │
└───────────────────────────────────────────────────────────────────────────┘
```

**Camera arrangement (8-camera example):**

```
Top of pendant (worn at chest level)
    ┌────────────────────────────────┐
    │ 315°  0°  45°                  │  ← Front hemisphere cameras (3)
    │   [Cam0] [Cam1] [Cam2]          │
    │                                │
    │ 270°        90°                │  ← Side cameras (2)
    │  [Cam7]      [Cam3]            │
    │                                │
    │ 225°  180°  135°               │  ← Rear hemisphere cameras (3)
    │   [Cam6] [Cam5] [Cam4]         │
    └────────────────────────────────┘
```

**Camera specifications per position:**

| Camera Position | FOV Requirement | Purpose |
|-----------------|-----------------|---------|
| Front (0°, ±45°) | ≥ 55° each | Forward visual capture, primary SLAM input |
| Side (90°, 270°) | ≥ 55° each | Lateral awareness, expansion of forward view |
| Rear (180°, ±45°) | ≥ 55° each | Rear awareness, behind-wearer monitoring |

**Sensor apertures:**

| Sensor | Aperture Type | Count | Position |
|--------|---------------|-------|----------|
| Camera array | Optical window | 4-8 | Distributed around outer arc |
| mmWave radar | RF-transparent window | 2-4 | Distributed for 360° coverage |
| IMU array | None | N/A | Internal |
| Microphone array | Acoustic ports | 4-8 | Along arc sides |
| Environmental | Ventilation | Multiple | Bottom-facing surfaces |

**Attachment:**
- Chain passes through end caps or loops around arc
- Two attachment points recommended for stability
- Balance: Weight distribution must prevent rotation
- Optional: Dedicated battery module on chain for extended runtime

**Thermal management (CRITICAL):**

| Source | Heat Generation | Mitigation |
|--------|-----------------|------------|
| Vision processor | 1-2 W | Thermal vias, heat spreader, air gap |
| Cameras (8 active) | 0.8-2 W | Duty cycling, selective activation |
| mmWave radar arrays | 0.3-0.5 W | Low power, acceptable |
| Total active | 2-4.5 W | Requires active management |

**Thermal design:**
- Thermal vias from vision processor to enclosure
- Thin thermal pad between PCB and outer arc (heat spreader function)
- Air gap between PCB and inner arc (silicone pad)
- Maximum inner arc temperature: 42°C
- Maximum outer arc temperature: 45°C (air side)
- Duty cycling for sustained operation

**Extended runtime power input:**
- Conductive chain traces (optional variant)
- Dedicated power cable from belt (tactical variant)
- Enables unlimited operation when powered from belt

**IP Rating:** IP67 (with proper sealing of camera windows and acoustic ports)

**Charging options:**
- Magnetic POGO dock (lays flat in dock)
- USB-C port in end cap
- Qi wireless (compatibility depends on metal thickness)

### 4.3 Variant C — Medallion (Premium)

**Target form:** Larger circular or hexagonal medallion with substantial profile.

**Dimensions:**
- Diameter: 60 mm
- Thickness: 20-25 mm
- Target weight: 80-150 g
- Battery: 800-1500 mAh

**Enclosure construction:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Medallion Pendant Enclosure                  │
├─────────────────────────────────────────────────────────────────┤
│  Front Face                                                      │
│  - Material: Titanium or surgical steel                          │
│  - Design: Decorative bezel with integrated sensor windows      │
│  - Window: Optical-grade polycarbonate for camera/LiDAR        │
├─────────────────────────────────────────────────────────────────┤
│  Side Walls                                                      │
│  - Thickness: 20-25 mm total                                     │
│  - Features: Control buttons, status LEDs, microphone ports     │
├─────────────────────────────────────────────────────────────────┤
│  Back Face                                                       │
│  - Material: Titanium with silicone pad                          │
│  - Features: Charging contacts, optional hardware switch        │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                        │
│  - PCB: Larger area, supports higher-compute processor          │
│  - Battery: 800-1500 mAh LiPo                                   │
│  - Sensors: mmWave, IMU, 6-element microphone array, LiDAR      │
│  - Optional: Structured light depth sensor                      │
└─────────────────────────────────────────────────────────────────┘
```

**Premium features:**
- Decorative metalwork options (engraving, inlay)
- Multiple finish options (brushed, polished, matte, PVD coating)
- Optional precious metal plating (gold, platinum — user-sourced)
- User-selectable camera resolution (2 MP to 12 MP)
- Hardware kill switch for camera (user-installed)

**IP Rating:** IP67

### 4.4 Variant D — Tactical/Extended Runtime

**Target form:** Larger profile medallion optimized for extended operation.

**Dimensions:**
- Diameter: 70-80 mm
- Thickness: 20-30 mm
- Target weight: 100-180 g
- Battery: 1500-2000 mAh (internal) + external power input

**Key features:**
- Integrated battery pack in enclosure
- External power input connector (compatible with belt battery)
- Active cooling vents (thermal management for continuous operation)
- Ruggedized construction for professional use
- Quick-swap battery option (in some designs)

**Thermal management:**
- Active ventilation slots
- Larger surface area for heat dissipation
- Option for active cooling (mini fan — requires larger enclosure)
- Designed for sustained 360° operation

**Use case:** Security personnel, journalists, users requiring all-day 360° awareness

### 4.5 Variant E — Event-Enhanced

**Target form:** Modified 360° pendant with hybrid camera array.

**Dimensions:** Same as Variant B

**Key difference:**
- 6 conventional cameras + 2 event cameras
- Event cameras positioned at key angles (typically forward ± 30°)
- Provides both visual capture and extreme velocity detection

**Sensor apertures:**
- Conventional cameras: Optical windows (6 positions)
- Event cameras: Optical windows optimized for event sensor wavelength
- Both window types can use same material (polycarbonate)

---

## 5. Bracelet Node Enclosures

### 5.1 Overview

**Target form:** Wristwatch-style band with dorsal module.

**Design philosophy:** Resembles a fitness tracker or slim smartwatch, not overtly technological.

### 5.2 Enclosure Construction

```
┌─────────────────────────────────────────────────────────────────┐
│                       Bracelet Node Enclosure                    │
├─────────────────────────────────────────────────────────────────┤
│  Dorsal Case (outer face, away from wrist)                      │
│  - Dimensions: 40 mm × 35 mm × 8-12 mm                          │
│  - Material: Titanium or surgical steel                          │
│  - Finish: Brushed, polished, or matte                           │
│  - Features: mmWave window (outward-facing), optional camera    │
├─────────────────────────────────────────────────────────────────┤
│  Ventral Case (inner face, skin contact)                        │
│  - Material: Medical-grade silicone or titanium                 │
│  - Features: Haptic actuator contact point, charging contacts  │
├─────────────────────────────────────────────────────────────────┤
│  Band Lugs                                                      │
│  - Width: 22 mm standard                                        │
│  - Type: Quick-release spring bar compatible                    │
│  - Compatible with third-party watch bands                      │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - PCB: 35 mm × 30 mm maximum                                   │
│  - Battery: 150-500 mAh (depending on variant)                  │
│  - Sensors: mmWave radar, IMU, haptic driver                   │
│  - Optional: Camera module (forward-facing through dorsal)      │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Sensor Apertures

| Sensor | Aperture | Position | Material |
|--------|----------|----------|----------|
| mmWave radar | RF-transparent window | Dorsal face (outward) | Polycarbonate, no metal |
| IMU | None | Internal | N/A |
| Haptic actuator | Vibration coupling | Ventral face | Silicone pad contact |
| Camera (optional) | Optical window | Dorsal face | Clear polycarbonate |

### 5.4 Haptic Integration

**Actuator type:** LRA (Linear Resonant Actuator) or ERM (Eccentric Rotating Mass)

**Placement:** Center of dorsal case, vibration transmitted through entire enclosure

**Driver:** TI DRV2605L with pre-loaded haptic patterns

**Human factors:**
- Vibration should be felt clearly on dorsal wrist
- Should NOT be uncomfortable or startling at maximum intensity
- Directional encoding via pattern variation, not multiple actuors

### 5.5 Charging

**Options:**
- Magnetic POGO pins on ventral face
- Qi wireless charging (compatible with standard charging pads)
- USB-C port (if form factor allows)

### 5.6 Band Options

| Band Type | Material | Attachment |
|-----------|----------|------------|
| Silicone sport band | Medical-grade silicone | 22 mm spring bars |
| Leather band | Genuine leather, hypoallergenic lining | 22 mm spring bars |
| Metal bracelet | Titanium links | 22 mm spring bars |
| Nylon strap | Woven nylon, hypoallergenic backing | 22 mm spring bars |

### 5.7 IP Rating

**IP67** — 1 m submersion for 30 minutes. Suitable for showering, swimming, rain.

### 5.8 Weight Targets

| Configuration | Battery | Weight (target) |
|---------------|---------|-----------------|
| Minimal | 150 mAh | 25-35 g |
| Standard | 200-300 mAh | 30-45 g |
| Extended (+ camera) | 400-500 mAh | 40-60 g |

---

## 6. Belt Node Enclosures

### 6.1 Overview

**Design philosophy:** The belt node is the central hub — larger form factor is acceptable because it's worn at the waist. Thermal management is critical due to higher compute power.

### 6.2 Variant A — MCU-Class (Minimal)

**Target form:** Slim belt buckle or small clip pack.

**Dimensions:**
- Buckle: 70 mm × 50 mm × 15 mm
- Weight: 100-150 g
- Battery: 2000 mAh

**Enclosure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   MCU Belt Node Enclosure                        │
├─────────────────────────────────────────────────────────────────┤
│  Front Face (buckle exterior)                                   │
│  - Material: Brushed aluminum or titanium                       │
│  - Features: Status LED, optional display                       │
├─────────────────────────────────────────────────────────────────┤
│  Rear Face (body contact)                                       │
│  - Material: Aluminum or titanium with thermal pad              │
│  - Features: BLE antenna (internal), USB-C port                │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - Compute: STM32H7-class MCU                                   │
│  - Battery: 2000 mAh Li-Ion                                     │
│  - Sensors: IMU, optional mmWave, environmental                │
│  - Radios: BLE coordinator, WiFi module                         │
│  - Storage: microSD card slot                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Thermal:**
- MCU power: 200-500 mW typical
- Thermal management: Passive sufficient
- Maximum skin-contact temperature: 40°C

**IP Rating:** IP54 (splash resistant, not submersible)

### 6.3 Variant B — Linux SoM (Standard)

**Target form:** Belt pack or larger buckle.

**Dimensions:**
- Pack: 100 mm × 70 mm × 25 mm
- Weight: 150-250 g
- Battery: 5000-7000 mAh

**Enclosure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   Linux SoM Belt Node Enclosure                  │
├─────────────────────────────────────────────────────────────────┤
│  Front Face                                                     │
│  - Material: Aluminum (heat spreading)                          │
│  - Features: Status LEDs, ventilation slots, optional display  │
├─────────────────────────────────────────────────────────────────┤
│  Rear Face (body contact)                                       │
│  - Material: Aluminum with thermal isolation layer              │
│  - Features: Silicone padding for comfort                       │
│  - Maximum temperature: 42°C (with thermal throttling)         │
├─────────────────────────────────────────────────────────────────┤
│  Side Faces                                                     │
│  - Features: USB-C port, SIM card slot (if cellular),          │
│    microSD slot, WiFi/cellular antenna connectors              │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - Compute: Raspberry Pi CM4 or NXP i.MX 8M Plus               │
│  - Battery: 5000-7000 mAh Li-Ion (single or dual pack)         │
│  - Sensors: IMU, mmWave, environmental                          │
│  - Radios: WiFi (SoM integrated), BLE coordinator, cellular    │
│  - Storage: NVMe SSD or SD card                                 │
│  - Thermal: Heat spreader, thermal vias, monitoring thermistor │
└─────────────────────────────────────────────────────────────────┘
```

**Thermal management (CRITICAL):**

| Power Scenario | Heat Generated | Mitigation |
|----------------|----------------|------------|
| Sparse tracking only | 1-2 W | Passive dissipation |
| Dense SLAM active | 2-4 W | Active ventilation, throttling |
| Full streaming | 3-5 W | Throttling, duty cycling |
| Worst case (everything active) | 4-7 W | Aggressive throttling, user warning |

**Thermal design features:**
- Thermal pad between SoM and aluminum enclosure
- Ventilation slots on front and sides
- NTC thermistor for firmware monitoring
- Firmware throttles SLAM/camera above 40°C
- Maximum sustained power: ~4 W without throttling

**Cellular module integration:**

```
Cellular Module Compartment (inside enclosure)
├── LTE/5G module with antenna connector
├── nano-SIM card slot (accessible from side or rear)
├── Thermal isolation from main SoM
└── U.FL connector to external antenna (on enclosure edge)
```

**Antenna placement:**

```
Belt Node Enclosure (Top View)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   [Cellular Antenna]                    [WiFi Antenna]          │
│   (Exterior edge, U.FL connector)       (Exterior edge)         │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐      │
│   │              Internal Electronics                   │      │
│   │                                                     │      │
│   │   [BLE Antenna]         [BLE Antenna]               │      │
│   │   (Interior, toward body)                           │      │
│   │                                                     │      │
│   └─────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**RF design rationale:**
- Cellular/WiFi antennas on exterior: better propagation, away from body absorption
- BLE antennas on interior: body is the communication medium for BAN
- Physical separation reduces coupling between 2.4 GHz radios

**IP Rating:** IP54

### 6.4 Variant C — High-Performance

**Target form:** Larger belt pack with active cooling capability.

**Dimensions:**
- Pack: 120 mm × 80 mm × 30 mm
- Weight: 200-300 g
- Battery: 7000 mAh (dual pack)

**Additional features:**
- Higher-performance SoM (Qualcomm SA8155P or i.MX 8M Plus with NPU)
- Optional active cooling (micro fan with filtered intake)
- Higher sustained compute power
- Multiple high-speed interfaces for 360° pendant data

**Thermal:**
- Designed for sustained 5-8 W operation
- Active cooling optional
- Larger thermal mass and surface area

### 6.5 Variant D — ESP32-Based

**Target form:** Minimal belt buckle.

**Dimensions:**
- Buckle: 60 mm × 40 mm × 15 mm
- Weight: 80-120 g
- Battery: 2000 mAh

**Features:**
- ESP32-S3 MCU with integrated WiFi
- WiFi used ONLY as BAN transport (architecture constraint)
- No cellular, no external antenna for WiFi
- Suitable for minimal deployments

### 6.6 Variant E — Extended Runtime/Hot-Swappable

**Target form:** Dual-battery belt pack.

**Dimensions:**
- Pack: 130 mm × 90 mm × 35 mm
- Weight: 250-350 g (with both batteries)
- Battery: 2 × 5000-7000 mAh (hot-swap capable)

**Key features:**

```
┌─────────────────────────────────────────────────────────────────┐
│               Hot-Swappable Battery Architecture                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Battery Bay A]          [Battery Bay B]                      │
│   Status: ACTIVE           Status: STANDBY                      │
│   ╔══════════════╗         ╔══════════════╗                    │
│   ║   Battery    ║         ║   Battery    ║                    │
│   ║   Pack A     ║         ║   Pack B     ║                    │
│   ║  5000 mAh    ║         ║  5000 mAh    ║                    │
│   ╚══════════════╝         ╚══════════════╝                    │
│                                                                  │
│   [Power Path Controller]                                        │
│   - Seamlessly switches between batteries                        │
│   - Can swap Bay B while powered by Bay A                       │
│   - Zero-downtime operation                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Use case:** 24/7 operation for security, shift workers, continuous monitoring.

**IP Rating:** IP54

---

## 7. Anklet Node Enclosures

### 7.1 Overview

**Target form:** Slim sports band or jewelry anklet with sensor module.

**Design philosophy:** Low-profile, unobtrusive, stays secure during walking and running.

### 7.2 Enclosure Construction

```
┌─────────────────────────────────────────────────────────────────┐
│                       Anklet Node Enclosure                      │
├─────────────────────────────────────────────────────────────────┤
│  Sensor Module                                                   │
│  - Dimensions: 40 mm × 30 mm × 12 mm                            │
│  - Material: Surgical steel with silicone strap                 │
│  - Position: Upper ankle (above ankle bone)                    │
├─────────────────────────────────────────────────────────────────┤
│  Front Face (forward-facing)                                    │
│  - Features: ToF/LiDAR window, optional camera window          │
│  - Material: Optical polycarbonate                              │
├─────────────────────────────────────────────────────────────────┤
│  Rear Face (skin contact)                                       │
│  - Material: Medical-grade silicone pad                         │
│  - Features: Haptic contact point, charging contacts           │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - PCB: 35 mm × 25 mm maximum                                   │
│  - Battery: 300-500 mAh                                         │
│  - Sensors: ToF/LiDAR, high-dynamic-range IMU (5-10 g)         │
│  - Optional: mmWave radar, small camera                        │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 Sensor Apertures

| Sensor | Aperture | Position | Notes |
|--------|----------|----------|-------|
| ToF/LiDAR | Optical window | Forward-facing | Measures ground clearance, obstacles |
| IMU | None | Internal | High-dynamic-range for heel-strike |
| mmWave (optional) | RF window | Forward-facing | Motion detection |
| Camera (optional) | Optical window | Forward/downward | Ground-level visual |
| Haptic | Vibration coupling | Rear face | Alerts, feedback |

### 7.4 IMU Considerations

**Critical requirement:** IMU must handle heel-strike impacts (5-10 g)

| IMU Spec | Requirement |
|----------|-------------|
| Accelerometer range | ±16 g minimum, ±32 g preferred |
| Gyroscope range | ±2000 dps |
| Sample rate | ≥ 200 Hz for gait analysis |
| Onboard processing | Preferred (reduces bandwidth) |

### 7.5 Gait Analysis Integration

Anklet is the primary gait analysis node:
- Detects step frequency (cadence)
- Measures heel-strike impact
- Identifies gait asymmetry
- Predicts stumble precursors

**Firmware processing:** Gait events detected on-node, transmitted as events (not raw IMU data).

### 7.6 Strap Options

| Strap Type | Material | Features |
|------------|----------|----------|
| Silicone sport | Medical-grade silicone | Secure, washable |
| Elastic band | Elastic with hypoallergenic liner | Adjustable |
| Magnetic clasp | Silicone with magnetic closure | Easy on/off |
| Jewelry anklet | Chain with safety clasp | Aesthetic priority |

### 7.7 IP Rating

**IP67** — Designed for water exposure, showering, rain.

### 7.8 Weight Targets

| Configuration | Battery | Weight |
|---------------|---------|--------|
| Minimal | 300 mAh | 40-60 g |
| Extended | 400-500 mAh | 50-100 g |

---

## 8. Eyewear Node Enclosures

### 8.1 Clip-On Design (Variants A, B)

**Target form:** Small sensor package that clips to existing eyeglass frames.

**Dimensions:**
- Module: 30 mm × 20 mm × 8 mm
- Weight: < 8 g (target < 5 g for comfort)
- Battery: 50-80 mAh

**Enclosure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Eyewear Clip-On Enclosure                     │
├─────────────────────────────────────────────────────────────────┤
│  Front Face (forward-facing)                                    │
│  - Features: Event camera window, optional conventional camera │
│  - Material: Lightweight plastic (ABS, PETG)                   │
├─────────────────────────────────────────────────────────────────┤
│  Clip Mechanism                                                 │
│  - Material: Flexible plastic or rubber                         │
│  - Type: Spring clip or friction fit                            │
│  - Requirement: Must not damage lens coatings                  │
├─────────────────────────────────────────────────────────────────┤
│  Internal                                                       │
│  - PCB: 25 mm × 15 mm maximum                                   │
│  - Battery: 50-80 mAh (small LiPo)                              │
│  - Sensors: Event camera, IMU, optional conventional camera    │
│  - Radio: BLE to belt node                                      │
└─────────────────────────────────────────────────────────────────┘
```

**Compatibility:**
- Fits most eyeglass frame bridges
- Does not interfere with prescription lenses
- Center of gravity should be near frame

**IP Rating:** IPX4 (sweat and splash resistant)

### 8.2 Frame-Integrated Design (Variant C)

**Target form:** Custom or semi-custom glasses frames with embedded electronics.

**Dimensions:**
- Temple arm segments: 60 mm × 4 mm × 3 mm (each arm)
- Bridge electronics: 25 mm × 15 mm × 8 mm
- Weight: 25-40 g (complete eyewear)
- Battery: 80-150 mAh (distributed in temples)

**Enclosure:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  Frame-Integrated Eyewear                        │
├─────────────────────────────────────────────────────────────────┤
│  Frame Front (bridge + lens surround)                           │
│  - Material: Titanium, acetate, or plastic                      │
│  - Features: Forward camera (bridge), optional side cameras    │
│  - Must accommodate prescription lenses                         │
├─────────────────────────────────────────────────────────────────┤
│  Temple Arms (left and right)                                   │
│  - PCB segments embedded in each temple                         │
│  - Features: Battery, IMU, haptic (one side), BLE antenna      │
│  - Thickness increase: +2-4 mm over standard frames            │
├─────────────────────────────────────────────────────────────────┤
│  Camera Configuration                                           │
│  - Forward camera: Bridge position                              │
│  - Side cameras: Temple arm junction points                     │
│  - Total cameras: 1-3 depending on variant                     │
├─────────────────────────────────────────────────────────────────┤
│  Charging                                                       │
│  - Magnetic POGO on temple tip                                  │
│  - Or: USB-C on temple (if form factor allows)                 │
└─────────────────────────────────────────────────────────────────┘
```

**Frame material options:**

| Material | Weight | Aesthetic | Cost |
|----------|--------|-----------|------|
| Titanium | Lightest | Premium | Highest |
| Acetate | Moderate | Fashion | Moderate |
| Plastic/TR-90 | Lightest | Sport | Lowest |
| Stainless steel | Heavier | Classic | Moderate |

**Prescription compatibility:**
- Must allow for prescription lens insertion
- Bridge camera placement must not interfere with lens mounting
- Frame front dimensions must follow optical standards

**IP Rating:** IPX4

### 8.3 Headband Design (Alternative to Frame-Integrated)

**Target form:** Headband with central forehead module and temporal extensions.

**Dimensions:**
- Headband: Adjustable, fits head circumference 50-65 cm
- Forehead module: 50 mm × 30 mm × 15 mm
- Weight: 30-50 g
- Battery: 100-150 mAh

**Advantages over frame-integrated:**
- Easier to prototype
- No prescription lens constraints
- More space for electronics and battery
- Can be worn over existing glasses

**Disadvantages:**
- More visible
- Less "everyday" aesthetic
- May interfere with hats, helmets

---

## 9. Thermal Management — Detailed Specifications

### 9.1 Thermal Constraints for Wearables

**Maximum temperatures:**

| Surface | Maximum Temperature | Reason |
|---------|---------------------|--------|
| Skin contact | 42°C | Discomfort threshold |
| Air-side surface | 50°C | Burn risk |
| Internal PCB | 85°C | Component reliability |

### 9.2 Thermal Design Strategies

**Strategy 1: Passive Dissipation**

| Node | Applicability | Implementation |
|------|---------------|----------------|
| Pendant Standard | Primary | Thermal vias, air gap |
| Bracelet | Primary | Metal case as heat spreader |
| Anklet | Primary | Metal case, limited power |
| Eyewear | Primary | Very low power, plastic dissipation |

**Strategy 2: Thermal Spreading**

| Node | Applicability | Implementation |
|------|---------------|----------------|
| Belt Linux SoM | Primary | Aluminum enclosure as heat sink |
| 360° Pendant | Primary | Metal arc as heat spreader |

**Strategy 3: Active Ventilation**

| Node | Applicability | Implementation |
|------|---------------|----------------|
| Belt High-Perf | Optional | Small fan with filtered intake |
| Belt Hot-Swap | Optional | Larger enclosure allows airflow |

**Strategy 4: Duty Cycling**

| Node | Applicability | Implementation |
|------|---------------|----------------|
| All nodes | Universal | Firmware throttles above temperature threshold |

### 9.3 Thermal Materials

| Material | Thermal Conductivity | Application |
|----------|---------------------|-------------|
| Aluminum (6061) | ~167 W/m·K | Enclosure shells, heat spreaders |
| Copper | ~400 W/m·K | Internal heat pipes (rare) |
| Thermal pad (silicone) | 1-6 W/m·K | Component to enclosure interface |
| Thermal vias | Variable | PCB heat transfer |
| Silicone pad (skin contact) | ~0.2 W/m·K | Thermal isolation from skin |

### 9.4 Thermal Monitoring

All nodes with significant power should include thermal monitoring:

| Component | Interface | Purpose |
|-----------|-----------|---------|
| NTC thermistor | ADC | PCB temperature monitoring |
| MCU internal temp | Internal ADC | MCU die temperature |
| Battery temp sensor | Fuel gauge IC | Battery safety |

**Firmware actions on over-temperature:**

| Temperature | Action |
|-------------|--------|
| 40°C (skin contact) | Reduce SLAM rate, reduce camera frame rate |
| 45°C (skin contact) | Disable SLAM, reduce sensing duty cycle |
| 50°C (skin contact) | Alert user, disable non-essential functions |
| 60°C (internal) | Emergency shutdown |

---

## 10. Water Resistance Specifications

### 10.1 IP Rating Definitions

| Rating | Solid Ingress | Water Ingress | Test Method |
|--------|---------------|---------------|-------------|
| IP54 | Dust protected | Splash resistant | Water spray |
| IP55 | Dust protected | Water jets | 6.3 mm nozzle, low pressure |
| IP65 | Dust tight | Water jets | 6.3 mm nozzle, low pressure |
| IP67 | Dust tight | 1 m immersion | 30 minutes submerged |
| IP68 | Dust tight | >1 m immersion | Manufacturer specified depth/duration |

### 10.2 IP Rating by Node

| Node | Rating | Rationale |
|------|--------|-----------|
| Pendant Standard | IP67 | May be worn in rain, shower, accidental submersion |
| Pendant 360° | IP67 | Same rationale |
| Pendant Medallion | IP67 | Same rationale |
| Pendant Tactical | IP67 | Same rationale |
| Bracelet | IP67 | Exposure to hand washing, rain, sweat |
| Belt MCU | IP54 | Partial protection, not submersible |
| Belt Linux SoM | IP54 | Same |
| Belt High-Perf | IP54 | Same |
| Belt Hot-Swap | IP54 | Same |
| Anklet | IP67 | Exposure to rain, puddles, sweat |
| Eyewear Clip-on | IPX4 | Sweat, splash; not submersible |
| Eyewear Frame-integrated | IPX4 | Same |

### 10.3 Sealing Methods

| Method | IP Rating | Components |
|--------|-----------|------------|
| Gasket (silicone) | IP67 | Enclosure seams, case backs |
| O-ring (silicone, Viton) | IP67 | Button shafts, connectors |
| PTFE membrane | IP67 (acoustic) | Microphone ports |
| Epoxy potting | IP68 | Connector backshells |
| UV-cure adhesive | IP67 | Window bonding |
| Thread sealant | IP65 | Threaded connections |
| Labyrinth seal | IP55 | Ventilation paths |

---

## 11. Charging Mechanisms

### 11.1 Charging Options by Node

| Node | Charging Methods | Power Source |
|------|------------------|--------------|
| Pendant Standard | POGO dock, USB-C, Qi wireless | 5V USB charger |
| Pendant 360° | POGO dock, USB-C, Belt power input | 5V USB or belt battery |
| Pendant Medallion | USB-C, Qi wireless | 5V USB charger |
| Bracelet | POGO pins (on wrist), Qi wireless | 5V USB charger |
| Belt (all variants) | USB-C PD | 5-20V USB-C charger |
| Anklet | POGO pins, Qi wireless | 5V USB charger |
| Eyewear Clip-on | POGO pins, USB-C (if form factor allows) | 5V USB charger |
| Eyewear Frame | Magnetic POGO on temple | 5V USB charger |

### 11.2 Charging Time Targets

| Node | Battery | Charging Rate | Full Charge Time |
|------|---------|----------------|-------------------|
| Pendant Standard | 300 mAh | 150 mA | 2 hours |
| Pendant 360° | 800 mAh | 400 mA | 2 hours |
| Pendant Medallion | 1500 mAh | 750 mA | 2 hours |
| Bracelet | 400 mAh | 200 mA | 2 hours |
| Belt MCU | 2000 mAh | 1000 mA | 2 hours |
| Belt Linux SoM | 7000 mAh | 2000 mA | 3.5 hours |
| Anklet | 500 mAh | 250 mA | 2 hours |
| Eyewear | 80 mAh | 80 mA | 1 hour |

### 11.3 Charging Dock Design

**Universal pendant dock:**
- Flat cradle with POGO pins
- Compatible with all pendant variants
- LED indicator for charging status

**Bracelet charging band:**
- Dock with recessed POGO contacts
- Bracelet placed face-down in dock

**Belt charging dock:**
- Stand with USB-C PD
- Status display for battery level
- Optional: Second battery charging slot (for hot-swap variant)

---

## 12. Weight Distribution and Balance

### 12.1 Pendant Balance

**Standard pendant:**
- Single attachment point (bail)
- Center of gravity should be below attachment point
- Battery positioned in lower half of enclosure

**360° pendant:**
- Distributed weight along arc
- Two attachment points recommended for stability
- Prevents rotation around neck
- Balance: Left-right symmetry

**Medallion:**
- Larger battery can shift center of gravity
- May require counterbalance or wide bail

### 12.2 Bracelet Balance

- Weight on dorsal face should not cause rotation
- Strap tension keeps case stable on wrist
- Battery centered within case

### 12.3 Belt Balance

- Weight on hip is well-tolerated
- Rear clip pack: Weight over sacrum
- Buckle: Weight on front of hip
- Symmetrical design prevents belt rotation

### 12.4 Anklet Balance

- Module should stay centered on ankle
- Strap design prevents rotation
- Battery centered within module

### 12.5 Eyewear Balance

**Critical:** Unbalanced eyewear causes discomfort

- Temple battery: Balance left and right
- Bridge camera: Center of gravity remains near nose
- Clip-on: Must not tip glasses forward
- Total eyewear weight: < 30 g recommended for all-day wear

---

## 13. Human Factors — Summary

### 13.1 Weight Tolerances

| Node | Comfortable Weight | Maximum Weight | Duration |
|------|-------------------|----------------|----------|
| Pendant | < 50 g | < 100 g | All day |
| Bracelet | < 40 g | < 70 g | All day |
| Belt | < 200 g | < 400 g | All day |
| Anklet | < 70 g | < 120 g | All day |
| Eyewear | < 20 g | < 40 g | All day |

### 13.2 Temperature Tolerances

| Surface | Maximum Comfortable | Maximum Tolerable |
|---------|--------------------|--------------------|
| Skin contact | 38°C | 42°C |
| Air-side | 45°C | 50°C |

### 13.3 Motion Compatibility

| Node | Motion Requirements |
|------|---------------------|
| Pendant | Must not swing excessively during walking; stable during head movement |
| Bracelet | Must not interfere with hand movement; secure during exercise |
| Belt | Must not shift during bending, sitting, walking |
| Anklet | Must not interfere with walking, running; secure during movement |
| Eyewear | Must not slip during head movement; must not interfere with peripheral vision |

### 13.4 Clothing Compatibility

| Node | Clothing Considerations |
|------|-------------------------|
| Pendant | Works with necklines, scarves, jackets |
| Bracelet | Works with sleeves, gloves, typing |
| Belt | Works with belts, waistbands, sitting |
| Anklet | Works with pants, socks, shoes |
| Eyewear | Works with hats, helmets, headphones |

---

## 14. CAD File Inventory

### 14.1 Pendant Node CAD

```
mechanical/cad/pendant_node/
├── pendant_std_enclosure.step              # Standard flat pendant enclosure
├── pendant_std_internal.step               # Internal PCB mounting
├── pendant_std_cross_section.dwg           # 2D cross-section drawing
├── pendant_360_arc.step                    # 360° curved pendant arc
├── pendant_360_camera_mount.step           # Camera mounting detail
├── pendant_360_full_assembly.step          # Complete 360° pendant assembly
├── pendant_360_thermal.step                # Thermal design detail
├── pendant_medallion_enclosure.step        # Medallion variant enclosure
├── pendant_medallion_internal.step         # Internal component layout
├── pendant_tactical.step                   # Tactical/extended runtime variant
├── pendant_tactical_battery.step           # Extended battery module
└── pendant_event_enhanced.step             # Event-enhanced variant
```

### 14.2 Bracelet Node CAD

```
mechanical/cad/bracelet_node/
├── bracelet_case_22mm.step                 # Standard case, 22 mm lugs
├── bracelet_case_internal.step             # PCB and battery mounting
├── bracelet_dorsal_face.step               # Outer face with sensor windows
├── bracelet_ventral_face.step              # Inner face with haptic contact
├── bracelet_charging_contacts.step         # POGO pin detail
└── bracelet_antenna_placement.step         # BLE antenna positions
```

### 14.3 Belt Node CAD

```
mechanical/cad/belt_node/
├── belt_node_buckle.step                   # Buckle-style enclosure
├── belt_node_buckle_internal.step          # Internal layout
├── belt_node_clip.step                     # Rear clip pack enclosure
├── belt_node_clip_internal.step            # Internal layout
├── belt_node_thermal.step                  # Thermal design details
├── belt_node_antenna_placement.step        # Antenna positions (BLE, WiFi, Cellular)
├── belt_node_sim_slot.step                 # SIM card holder detail
├── belt_node_hotswap.step                  # Hot-swap battery variant
├── belt_node_hotswap_battery.step          # Battery bay detail
└── belt_node_connectors.step               # USB-C, SIM, antenna connectors
```

### 14.4 Anklet Node CAD

```
mechanical/cad/anklet_node/
├── anklet_module_enclosure.step            # Sensor module enclosure
├── anklet_module_internal.step             # PCB and battery mounting
├── anklet_strap.step                       # Strap design
├── anklet_strap_attachment.step           # Module-to-strap connection
└── anklet_sensor_windows.step             # ToF/camera window detail
```

### 14.5 Eyewear Node CAD

```
mechanical/cad/eyewear_node/
├── eyewear_clipon.step                     # Clip-on module
├── eyewear_clipon_internal.step            # PCB and battery
├── eyewear_clipon_mechanism.step          # Clip mechanism detail
├── eyewear_frame_arm_segment.step          # Temple arm PCB segment
├── eyewear_frame_bridge.step               # Bridge camera housing
├── eyewear_frame_full.step                 # Complete frame-integrated design
├── eyewear_headband.step                   # Headband variant
├── eyewear_headband_module.step            # Forehead module
└── eyewear_prescription_lens.step          # Prescription lens integration
```

### 14.6 Assembly and Installation CAD

```
mechanical/cad/assembly/
├── full_system_layout.step                 # All nodes in wearing position
├── pendant_chain_attachment.step           # Chain/bail detail
├── belt_pack_attachment.step               # Belt attachment detail
├── charging_dock_pendant.step              # Pendant charging dock
├── charging_dock_belt.step                 # Belt charging stand
└── shipping_case.step                      # Protective case for storage/transport
```

---

## 15. STL Exports for 3D Printing

```
mechanical/stl/
├── pendant_std_enclosure.stl
├── pendant_360_arc.stl
├── pendant_medallion_enclosure.stl
├── bracelet_case_22mm.stl
├── belt_node_buckle.stl
├── belt_node_clip.stl
├── anklet_module_enclosure.stl
├── eyewear_clipon.stl
└── eyewear_headband.stl
```

**Note:** STL exports are for prototyping only. Production enclosures should be manufactured from metal using machining, casting, or metal injection molding for jewelry-grade finish and durability.

---

## 16. Production Manufacturing Notes

### 16.1 Prototyping

| Method | Material | Cost | Quality | Turnaround |
|--------|----------|------|---------|------------|
| FDM 3D printing | PLA, PETG, ABS | Low | Low | Hours |
| SLA 3D printing | Resin | Medium | Medium | Days |
| CNC machining | Aluminum, plastic | High | High | Days |
| Soft tooling | Silicone, urethane | Medium | Medium | Weeks |

### 16.2 Low-Volume Production

| Method | Material | Cost | Quality | Volume |
|--------|----------|------|---------|--------|
| CNC machining | Titanium, aluminum | High | Jewelry-grade | 10-100 units |
| Sheet metal forming | Stainless steel | Medium | Good | 50-500 units |
| Investment casting | Titanium, steel | Medium | Jewelry-grade | 50-1000 units |

### 16.3 High-Volume Production

| Method | Material | Cost | Quality | Volume |
|--------|----------|------|---------|--------|
| Metal injection molding | Stainless steel | High tooling, low part | Jewelry-grade | > 10,000 units |
| Die casting | Zinc, aluminum | High tooling, low part | Good | > 10,000 units |
| Stamping | Stainless steel | Medium tooling, low part | Good | > 5,000 units |

### 16.4 Finish Options

| Finish | Application | Method |
|--------|-------------|--------|
| Brushed | Metal surfaces | Linear abrasive |
| Polished | Metal surfaces | Multi-stage polishing |
| Bead blast | Metal surfaces | Glass bead media |
| Anodized | Aluminum | Electrochemical |
| PVD coating | All metals | Physical vapor deposition |
| Powder coat | Metal surfaces | Electrostatic spray + bake |
| Cerakote | Metal surfaces | Spray-on ceramic coating |

---

## 17. Quality Control

### 17.1 Visual Inspection

| Check | Specification |
|-------|---------------|
| Surface finish | No scratches, dents, or tool marks visible at 30 cm |
| Window clarity | No bubbles, scratches, or optical distortion |
| Seam alignment | < 0.5 mm gap or mismatch |
| Logo/branding | Properly positioned, legible |

### 17.2 Dimensional Inspection

| Check | Specification |
|-------|---------------|
| Overall dimensions | ± 0.5 mm |
| Window position | ± 0.3 mm |
| Button/keyhole position | ± 0.3 mm |
| Strap lug width | ± 0.1 mm |

### 17.3 Environmental Testing

| Test | Specification |
|------|---------------|
| IP rating test | Per IEC 60529 |
| Temperature cycling | -20°C to +50°C, 10 cycles |
| Humidity exposure | 85% RH, 48 hours |
| UV exposure | Per ASTM G154 |
| Salt spray | Per ASTM B117 (if outdoor variant) |

### 17.4 Wear Testing

| Test | Duration | Criteria |
|------|----------|----------|
| Abrasion resistance | 10,000 cycles | No visible wear |
| Drop test | 1 m onto concrete | No functional damage |
| Strap pull test | 50 N for 1 minute | No failure |
| Hinge/clasp cycling | 10,000 cycles | No failure |

---

## 18. Configuration Reference

### 18.1 Enclosure Configuration Options

```toml
[enclosure]
# Pendant configuration
pendant_variant = "standard"        # "standard" | "360_curved" | "medallion" | "tactical" | "event_enhanced"
pendant_finish = "brushed_titanium" # "brushed_titanium" | "polished_titanium" | "stainless_steel"
pendant_chain_type = "standard"    # "standard" | "heavy" | "custom"

# Bracelet configuration
bracelet_case_size = "standard"    # "slim" | "standard" | "extended"
bracelet_band_type = "silicone"    # "silicone" | "leather" | "metal" | "nylon"
bracelet_band_width_mm = 22

# Belt configuration
belt_style = "clip_pack"          # "buckle" | "clip_pack"
belt_finish = "brushed_aluminum"  # "brushed_aluminum" | "black_anodized"

# Anklet configuration
anklet_strap_type = "silicone"    # "silicone" | "elastic" | "magnetic"

# Eyewear configuration
eyewear_style = "clip_on"         # "clip_on" | "frame_integrated" | "headband"
eyewear_frame_material = "titanium" # For frame_integrated variant

# IP rating requirements
ip_rating = "IP67"                # IP54, IP55, IP65, IP67, IPX4

# Thermal
thermal_monitoring = true
max_skin_temperature_c = 42
```

---

## 19. Summary — Mechanical Specifications by Node

| Node | Dimensions | Weight | IP Rating | Battery | Thermal |
|------|------------|--------|-----------|---------|---------|
| Pendant Standard | 50×50×15 mm | 30-60 g | IP67 | 150-300 mAh | Passive |
| Pendant 360° | 120-160×15×10-15 mm arc | 50-120 g | IP67 | 400-1000 mAh | Active management |
| Pendant Medallion | 60×60×20-25 mm | 80-150 g | IP67 | 800-1500 mAh | Passive |
| Pendant Tactical | 70-80×70-80×20-30 mm | 100-180 g | IP67 | 1500-2000 mAh | Active ventilation |
| Bracelet | 40×35×8-12 mm | 25-60 g | IP67 | 150-500 mAh | Passive |
| Belt MCU | 70×50×15 mm | 100-150 g | IP54 | 2000 mAh | Passive |
| Belt Linux SoM | 100×70×25 mm | 150-250 g | IP54 | 5000-7000 mAh | Active |
| Belt Hot-Swap | 130×90×35 mm | 250-350 g | IP54 | 2×5000-7000 mAh | Active |
| Anklet | 40×30×12 mm | 40-100 g | IP67 | 300-500 mAh | Passive |
| Eyewear Clip-on | 30×20×8 mm | 5-15 g | IPX4 | 50-80 mAh | Passive |
| Eyewear Frame | Variable | 25-40 g | IPX4 | 80-150 mAh | Passive |

---

**End of Mechanical Specification — SENTINEL-WEAR Enclosures**
