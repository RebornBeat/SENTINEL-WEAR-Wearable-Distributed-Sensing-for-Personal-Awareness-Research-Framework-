# Hardware — SENTINEL-WEAR

This directory contains the hardware reference designs for SENTINEL-WEAR's sensing nodes: pendant, bracelet, belt, anklet, and (optional) eyewear form factors. Each node carries some subset of mmWave radar, IMU, microphone array, short-range LiDAR/ToF, environmental sensors, BAN radio, battery, and haptic actuator as appropriate to its form factor.

---

## Architectural Foundation — The Sole Gateway Principle

### Critical Architectural Constraint

**Only the Belt Node connects to external networks.** This is not a configuration choice — it is an architectural constraint enforced at both hardware and firmware levels.

**All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via Body-Area Network (BAN) to the belt node.** They have no WiFi, no cellular, no direct connection to the companion app, and no external network interface of any kind.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     SENTINEL-WEAR Hardware Connectivity Model                │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  ⚠️  BELT NODE:     WiFi + Cellular + BLE (to external devices)             │
│  ⚠️  ALL OTHER NODES: BAN only (BLE/UWB to belt, no external connectivity)   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why This Architecture?

| Constraint | WiFi on Wearable | BLE on Wearable | UWB on Wearable |
|------------|------------------|-----------------|-----------------|
| **Power** | 300-1000+ mW active | 5-15 mW active | 50-150 mW active |
| **Heat** | Significant thermal load | Minimal | Moderate |
| **Form factor impact** | Requires larger battery, antenna space | Minimal | Moderate |
| **Wearability** | Breaks jewelry form factor | Preserves jewelry form | Compatible |
| **Determinism** | Variable latency, contention | Consistent, schedulable | Excellent timing |
| **Antenna size** | Requires substantial antenna | Very small antenna | Small antenna |

**Conclusion:** WiFi on jewelry-form-factor nodes destroys the form factor through power, thermal, and size requirements. BLE and UWB are the only viable BAN technologies for jewelry-scale wearables.

### Regulatory Compliance

All nodes comply with:
- **FCC Part 15** (unlicensed RF operation in ISM bands)
- **CE RED** (EU Radio Equipment Directive)
- **ISED** (Canada) equivalent requirements
- **UL/IEC 60950-1** or **IEC 62368-1** for electrical safety

Cellular modules (belt node only) require carrier certification per specific module.

---

## Sensing-Only Scope

The hardware in this directory is **sensing-only**. No design in this directory implements:

- Active emitters above eye-safe educational classes.
- RF emitters operating outside FCC Part 15 / CE RED unregulated bands.
- Mechanical actuators capable of acting against any external object.
- Energy storage at densities or chemistries beyond standard wearable lithium-polymer.
- Any mechanism intended to provide protective function in the personal-protective-equipment sense.
- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

Researchers contributing hardware designs are asked to confirm in their pull requests that the contribution stays within these bounds. PRs proposing active-response hardware will be closed and the contributor referred to `docs/theory/future_research.md` for the appropriate research-domain context.

See `legal/compliance.md`, `legal/research_ethics.md`, and `legal/export_control_posture.md` for the project's full posture.

---

## Node Types and Hardware Variants

### Principle: Hardware Variants vs Configuration Options

**Hardware Variant:** Requires different PCB design, physical form factor, or core component selection. Documented as separate design in `schematic/` subdirectory.

**Configuration Option:** Same PCB design, different component population or firmware settings. Documented as options within a variant's README.

---

## Node Type 1: Pendant / Necklace Node

The highest-information-value node. Positioned at chest level for upper-hemisphere coverage.

### Hardware Variants

| Variant | Form Factor | Key Differentiators | Directory |
|---------|-------------|---------------------|-----------|
| **A — Standard Flat** | Flat medallion, 50×50×15mm max | mmWave + IMU + acoustic, optional camera | `schematic/pendant_node/` |
| **B — 360° Curved** | Curved arc pendant, 120-160mm arc | 4-8 camera array, 360° coverage | `schematic/360_pendant_node/` |
| **C — Medallion** | Larger circular, 60mm diameter × 20-25mm | Higher compute, higher-capacity battery | `schematic/pendant_node/` |
| **D — Tactical** | 70-80mm profile | Extended battery (up to 2000 mAh), belt power input | `schematic/pendant_node/` |
| **E — Event-Enhanced** | Based on B or C | Event cameras added for extreme velocity detection | `schematic/360_pendant_node/` |

### Configuration Options (Within Variants)

For Variants A, C, D:
| Option | Values | Implementation |
|--------|--------|----------------|
| Camera | None / Forward / Wide-angle | Footprint on PCB, optionally populated |
| UWB | Disabled / Enabled | Footprint on PCB, optionally populated |
| Battery capacity | 150 / 250 / 400 mAh | Different cell in same enclosure |
| Microphone count | 3 / 4 / 6 element | Different population |

For Variant B (360° Curved):
| Option | Values | Implementation |
|--------|--------|----------------|
| Camera count | 4 / 6 / 8 | Different population on flex PCB |
| Camera resolution | QVGA / VGA / 720p / 1080p | Same sensors, different mode |
| Stitching location | Pendant / Belt | Vision processor placement |
| UWB | Disabled / Enabled | For bandwidth augmentation |

### Connectivity

**All pendant variants:**
- BLE 5.x (mandatory) — control plane, metadata
- UWB (optional) — timing, ranging, moderate-bandwidth streaming
- **NO WiFi** — not present on any pendant variant
- **NO Cellular** — not present on any pendant variant
- All data routes via BAN to belt node → belt relays to companion app

### 360° Curved Pendant Architecture

The 360° pendant (Variant B) uses a **wired internal camera bus** architecture:

```
4-8 Cameras → MIPI CSI-2 (internal) → Vision Processor on Pendant
                                              │
                                              ├── Stitch on-pendant → Single 360° stream
                                              └── Encode (H.264/H.265)
                                                      │
                                                      ▼
                                                  UWB to Belt
                                                      │
                                                      ▼
                                              Belt → WiFi → Companion App
```

**Hardware synchronization:** All cameras share a single FSYNC line for simultaneous capture. PCB trace matching ensures < 10 ns skew between cameras.

**Bandwidth over UWB:** Supports 360° at 2K-2.5K resolution sustained. 4K requires WiFi (not available on pendant) or belt-assisted progressive quality strategies.

---

## Node Type 2: Bracelet Node (×2)

Forearm-hemisphere sensing and directional haptic output. Worn on both wrists.

### Hardware Design

**Single PCB design** supports all configurations. Different options achieved through component population and firmware configuration.

| Component | Options |
|-----------|---------|
| MCU | nRF5340, STM32WB55, Silicon Labs EFR32BG24 |
| mmWave Radar | Acconeer XR112 (60 GHz, ultra-compact) |
| IMU | BMI270 or ICM-42688-P |
| Haptic | TI DRV2605L driver + LRA actuator |
| Optional Camera | OV2640, HM01B0 (QVGA), or IMX219 |
| Optional UWB | Qorvo DW3000 footprint |
| Battery | 150-400 mAh (configurable) |

### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal / Standard / Extended | Different component population |
| Camera | None / Forward-facing | Optional module population |
| UWB | Disabled / Enabled | Optional module population |
| Battery | 150 / 250 / 400 mAh | Different cell sizes |

### Connectivity

- BLE 5.x (mandatory)
- UWB (optional)
- **NO WiFi**
- **NO Cellular**
- Data routes via BAN to belt node only

### Antenna Orientation

mmWave radar antenna must face outward (dorsal side of wrist) because body absorption significantly attenuates signals facing the wrist. Design places antenna on outer surface of bracelet case.

---

## Node Type 3: Belt Node — Primary Hub and Sole External Gateway

The central coordination unit. All other nodes communicate only with the belt. The belt is the **only node with external network connectivity**.

### Role

| Function | Belt Node Responsibility |
|----------|--------------------------|
| Compute hub | Runs fusion algorithms, PentaTrack, SLAM |
| Torso reference | IMU defines body-frame origin (+Y forward, +X right, +Z up) |
| BAN hub | Routes all inter-node communication |
| **External gateway** | **ONLY node with WiFi and Cellular** |
| App server | Embedded HTTP/WebSocket server for companion app |
| Recording manager | Aggregates recordings from all nodes |
| Media relay | Receives BAN streams, relays via WiFi/Cellular to app |

### Hardware Variants

| Variant | Compute Platform | Capability | Battery | Runtime |
|---------|------------------|------------|---------|---------|
| **A — MCU** | STM32H7 (Cortex-M7, 480 MHz) | Sparse tracking only | ~2000 mAh | 15-20 hrs (sparse) |
| **B — Linux SoM** | Raspberry Pi CM4 / NXP i.MX 8M Plus | Full SLAM, streaming | 5000-7000 mAh | 10-15 hrs (sparse), 5-8 hrs (active) |
| **C — High-Performance** | Qualcomm SA8155P or NXP i.MX 8M Plus with NPU | GPU-class SLAM, neural stitching | 5000-7000 mAh | 5-8 hrs (active) |
| **D — ESP32** | ESP32-S3 | Minimal deployment, 2-3 nodes | ~2000 mAh | 12-15 hrs |
| **E — Hot-Swappable** | Based on B or C | Dual battery bays, swap without downtime | 2× 5000 mAh | Unlimited with swaps |

### Connectivity (Belt Node Only)

| Interface | Purpose | Belt Only? |
|-----------|---------|------------|
| WiFi 802.11ac/ax | Primary companion app connection | **Yes** |
| Cellular (LTE/5G) | Remote access, emergency alerts | **Yes** |
| Bluetooth 5.3 | Local direct app connection (fallback) | **Yes** |
| USB-C | PC companion app, charging | **Yes** |
| BLE 5.x | BAN to other nodes | All nodes |
| UWB (optional) | BAN precision timing | Belt + optional on other nodes |

### Cellular Modules Supported

| Module | Technology | Speed | Use Case |
|--------|------------|-------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | Alerts, metadata |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | Compressed video |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12, eSIM | 300+ Mbps | Multi-carrier |

### Power Budget

| Mode | Active Functions | Power Draw | Runtime (5000 mAh) |
|------|------------------|------------|-------------------|
| Minimal | BAN hub, sparse tracking | ~500 mW | ~37 hours |
| Standard | BAN, sparse tracking, WiFi idle | ~1.4 W | ~13 hours |
| Full Active | BAN, dense SLAM, 360° stitching, streaming | ~4.5 W | ~4 hours |
| Remote Streaming | Full Active + cellular | ~5.3 W | ~3.5 hours |

### Thermal Management (Linux SoM Variants)

Sustained compute generates 3-7 W. Requirements:
- Thermal pad under SoM
- Vented or heat-spreader enclosure
- NTC thermistor for monitoring
- Firmware throttles SLAM frame rate above 40°C
- Maximum skin-contact temperature: 42°C

---

## Node Type 4: Anklet Node (×2)

Ground-plane sensing, gait analysis, lower-hemisphere coverage.

### Hardware Design

**Single PCB design** supports all configurations.

| Component | Options |
|-----------|---------|
| MCU | Nordic nRF5340 or Silicon Labs EFR32BG24 |
| Range sensor | VL53L5CX ToF (8×8, 6 m) or short-range LiDAR |
| IMU | BMI270 (high dynamic range for heel-strike: 5-10 g) |
| Haptic | TI DRV2605L + LRA |
| Optional mmWave | Acconeer XR112 |
| Optional Camera | HM01B0 (QVGA) or OV2640 |
| Optional UWB | DW3000 footprint |
| Battery | 300-500 mAh |

### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor set | Minimal (ToF only) / Extended (ToF + LiDAR + mmWave) | |
| Camera | None / Downward-facing | Ground obstacles |
| UWB | Disabled / Enabled | Recommended for gait sync |
| Battery | 300 / 500 mAh | |

### Gait Analysis Capabilities

On-node gait event detection transmits events (not raw IMU) to belt:
- Step frequency (cadence)
- Step regularity (coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right)
- Pre-stumble signature detection
- Gait phase: stance / swing / strike / push at ≥ 200 Hz

Raw IMU burst sent only on anomaly detection for detailed analysis.

### Connectivity

- BLE 5.x (mandatory)
- UWB (optional, recommended)
- **NO WiFi**
- **NO Cellular**

---

## Node Type 5: Eyewear Node (Optional)

Head-stabilized forward-hemisphere sensing, fast transient detection, SLAM visual odometry anchor.

### Hardware Variants

| Variant | Form Factor | Sensors | Weight | Battery |
|---------|-------------|---------|--------|---------|
| **A — Event-Only Clip-On** | 30×20×8 mm clip | Event camera + IMU | < 8 g | 50-80 mAh |
| **B — Event + Camera Clip-On** | Same as A | Event + conventional camera + IMU | < 10 g | 50-80 mAh |
| **C — Frame-Integrated** | Temple arm segments | Event + forward + side cameras + IMU | 20-40 g | 80-150 mAh |
| **D — Headband** | Forehead band | Full camera array + IMU | 30-50 g | 80-150 mAh or belt cable |

### Configuration Options

| Option | Values | Notes |
|--------|--------|-------|
| Sensor type | Event-only / Event + Camera | |
| Camera count | 1 / 2 / 3 | Forward + sides |
| UWB | Disabled / Enabled | For head-torso sync |
| Power source | Battery / Belt cable (Variant C/D) | Cable for extended runtime |

### SLAM Integration

- **Variant A:** Sparse optical flow only, limited SLAM contribution
- **Variant B:** Conventional camera provides keyframes for loop closure
- **Variant C/D:** Wide-baseline stereo between forward and side cameras enables depth estimation; 180°+ coverage prevents visual odometry failure during rotation

### Connectivity

- BLE 5.x (mandatory)
- UWB (optional)
- **NO WiFi**
- **NO Cellular**

---

## Body-Area Network (BAN) Architecture

### BLE 5.x — Primary BAN Transport

**Present on all nodes.** Lowest power, consistent latency.

| Capability | Specification |
|------------|---------------|
| Bandwidth | 500 Kbps - 2 Mbps (practical) |
| Latency | Configurable: 7.5 ms - 1000 ms connection interval |
| Power | 5-15 mW active, < 1 mW sleep |
| Range | ~10 m on-body |

**BLE scheduling:**
- Belt node = central, time master
- All other nodes = peripherals
- Time-slotted communication prevents collisions
- Priority: Belt IMU > Anklets > Pendant > Bracelets > Eyewear

### UWB — Precision Timing and Bandwidth Augmentation

**Optional on any node.** Higher power, higher capability.

| Capability | Specification |
|------------|---------------|
| Bandwidth | 4-6 Mbps sustained, 6.8 Mbps peak |
| Timing precision | Sub-nanosecond synchronization |
| Ranging precision | Centimeter-level distance measurement |
| Power | 50-150 mW active |

**UWB roles (configurable):**

| Role | Description | Power |
|------|-------------|-------|
| Timing only | Sub-ns sync, cm ranging | Lowest UWB power |
| Timing + Burst | Plus short data bursts | Moderate |
| Full bandwidth | Sustained streaming (single camera, 360° at 2K) | Highest |

**UWB supports:**
- All 8 pendant cameras at QVGA-VGA simultaneously
- Stitched 360° at 2K-2.5K resolution
- Single camera at 720p-1080p
- Tiered progressive quality strategies

**UWB does NOT support:**
- 4K 360° live streaming
- All cameras at 720p+ simultaneously
- Raw uncompressed sensor data streams

### Bandwidth Hierarchy Summary

```
BLE (500 Kbps - 2 Mbps)
  → Control plane, metadata, low-bandwidth sensor data
  → Always-on, all nodes
  → Power: 5-15 mW

UWB (4-6 Mbps sustained)
  → Precision timing, ranging
  → Moderate-bandwidth streaming (720p single camera, 2K 360°)
  → Optional, enabled per node
  → Power: 50-150 mW when active

WiFi (50-300+ Mbps) — BELT NODE ONLY
  → High-bandwidth streaming (4K 360°)
  → Primary companion app path
  → Power: 300-500 mW

Cellular (5-1000+ Mbps) — BELT NODE ONLY
  → Remote access when WiFi unavailable
  → Power: 150-1200 mW depending on module
```

---

## RF Coexistence

### Frequency Bands in Use

| Technology | Frequency Band | Potential Conflicts |
|------------|---------------|---------------------|
| BLE | 2.4 GHz ISM | WiFi 2.4 GHz |
| WiFi | 2.4 GHz, 5 GHz | BLE at 2.4 GHz |
| UWB | 3.1-10.6 GHz | None (separate band) |
| mmWave Radar | 60 GHz | None (separate band) |
| Cellular | Various (700 MHz - 3.5 GHz, 24-47 GHz 5G mmWave) | Minor at adjacent bands |

### Coexistence Strategies

1. **WiFi prefers 5 GHz band** — eliminates 2.4 GHz conflict with BLE
2. **Antenna separation on belt** — Cellular/WiFi antennas on exterior, BLE/UWB on interior toward body
3. **Time-division multiplexing** — If 2.4 GHz WiFi must be used, coordinate timing with BLE
4. **UWB is in separate band** — No conflict with BLE or WiFi

### Antenna Placement Strategy

```
Belt Node Enclosure (Top View)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   [Cellular Antenna]                    [WiFi Antenna]  │
│   (Exterior, separate from body)        (Exterior)      │
│                                                         │
│   ┌─────────────────────────────────────────────┐      │
│   │              PCB / Internal                 │      │
│   │                                             │      │
│   │   [BLE Antenna]         [BLE Antenna]       │      │
│   │   (Interior, toward body)                   │      │
│   │                                             │      │
│   └─────────────────────────────────────────────┘      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Power Architecture

### Power Sources

| Source | Nodes | Notes |
|--------|-------|-------|
| LiPo battery | All nodes | Standard wearable chemistry |
| USB-C charging | All nodes | 5V input |
| Belt power input | Pendant (tactical), Eyewear (frame-integrated) | Extended runtime |
| PoE | Not applicable (wearable) | |

### Power Domains

| Domain | Voltage | Usage |
|--------|---------|-------|
| V_BAT | 3.7-4.2V | Radio PA, haptic peak |
| V_SYS | 3.3V | MCU, sensors |
| V_RADIO | 3.3V | BLE/UWB radio |
| V_SENSOR | 3.3V / 1.8V | High-power sensors |
| V_CAM | 3.3V / 2.8V | Camera rail (gated) |

### Battery Safety

All LiPo cells: UL 1642 or IEC 62133 certified
- Overcharge cutoff: 4.25V
- Overdischarge cutoff: 3.0V
- Thermal cutoff: NTC thermistor, > 60°C during charge
- Short-circuit: PTC fuse or protection IC

### Power Estimates by Node

| Node Variant | Sleep (mW) | Active (mW) | Peak (mW) |
|--------------|------------|-------------|-----------|
| Pendant Standard | 10-50 | 200-500 | 800 |
| Pendant 360° | 50-100 | 2500-4500 | 6000 |
| Bracelet | 5-20 | 100-300 | 500 |
| Belt MCU | 20-50 | 300-800 | 2000 |
| Belt Linux SoM | 100-300 | 1000-5000 | 8000 |
| Anklet | 5-20 | 100-400 | 600 |
| Eyewear Clip-On | 5-10 | 100-300 | 500 |
| Eyewear Frame | 10-30 | 200-600 | 1000 |

---

## Materials and Human Factors

### Hypoallergenic Materials (Skin Contact)

All skin-contact surfaces comply with EN 1811:2011 nickel release requirements:

**Acceptable:**
- Grade 5 titanium (Ti-6Al-4V)
- 316L surgical stainless steel
- Medical-grade silicone (ISO 10993)
- Hard-coat anodized aluminum (enclosures only)

**Not Acceptable:**
- 6061/7075 aluminum (direct skin contact)
- Unplated brass
- Zinc die-cast

### Water Resistance

| Node | Minimum Rating | Test Method |
|------|----------------|-------------|
| Pendant | IP67 | 1 m submersion, 30 min |
| Bracelets | IP67 | 1 m submersion, 30 min |
| Belt Node | IP54 | Splash test |
| Anklets | IP67 | 1 m submersion, 30 min |
| Eyewear Clip-On | IPX4 | Sweat/splash |
| Eyewear Frame | IPX4 | Sweat/splash |

### Weight Targets

| Node | Target Weight | Notes |
|------|---------------|-------|
| Pendant Standard | 30-60 g | Comfortable for extended wear |
| Pendant 360° | 50-120 g | Heavier due to cameras/battery |
| Pendant Medallion | 80-150 g | Premium, larger profile |
| Bracelet | 25-60 g | Watch-like |
| Belt MCU | 100-150 g | Including battery |
| Belt Linux SoM | 200-300 g | Larger battery bank |
| Anklet | 40-70 g | Slim profile |
| Eyewear Clip-On | < 8 g | Must not burden glasses |
| Eyewear Frame | 20-40 g | Integrated electronics |

---

## Directory Structure

```
hardware/
├── schematic/
│   ├── pendant_node/           # Variants A, C, D (standard, medallion, tactical)
│   │   ├── variant_a_standard/
│   │   ├── variant_c_medallion/
│   │   ├── variant_d_tactical/
│   │   └── README.md
│   ├── 360_pendant_node/       # Variant B (360° curved) — separate node type
│   │   ├── rigid_flex_design/
│   │   ├── camera_array/
│   │   └── README.md
│   ├── bracelet_node/          # Single design, multiple configs
│   │   └── README.md
│   ├── belt_node/              # Variants A, B, C, D, E
│   │   ├── variant_a_mcu/
│   │   ├── variant_b_linux_som/
│   │   ├── variant_c_high_perf/
│   │   ├── variant_d_esp32/
│   │   ├── variant_e_hotswap/
│   │   └── README.md
│   ├── anklet_node/            # Single design, multiple configs
│   │   └── README.md
│   ├── eyewear_node/           # Variants A, B, C
│   │   ├── variant_a_clip_on/
│   │   ├── variant_b_clip_camera/
│   │   ├── variant_c_frame/
│   │   └── README.md
│   └── hardware_config.md      # Interface standards, pin maps
├── testing/
│   └── test_jig_pcb/           # Production test jig
│       └── README.md
├── bom/                        # Bill of materials templates
└── README.md                   # This file
```

---

## Testing and Quality Control

### Test Jig

Universal Test Station (UTS) validates all node variants:

**Carrier board:**
- Test controller MCU (STM32H7 or RP2040)
- Host SBC (Raspberry Pi 5)
- Programmable power supply
- Current sensing (per rail)
- RF shield for radar isolation
- BAN golden node reference

**Node-specific test shields:**
- Pendant Shield: Radar reflector, acoustic speaker, camera target
- 360° Pendant Shield: Multi-position LED array, stitching target, sync analyzer
- Bracelet Shield: Radar absorber, haptic resonance fixture
- Belt Shield: WiFi AP, cellular simulator, thermal load fixture
- Anklet Shield: ToF target, IMU impact simulator
- Eyewear Shield: Event LED strobe, IMU gimbal

### Mandatory Tests (All Nodes)

1. Power-on self-test
2. Rail verification (±5%)
3. Sensor enumeration
4. IMU gravity vector (9.81 ± 0.2 m/s² at rest)
5. BAN connectivity to belt node (< 200 ms)
6. Battery level verification
7. Firmware version match

### Cellular Tests (Belt Node Only)

1. SIM detection
2. Module AT command response
3. Network registration (with live SIM)
4. Signal strength
5. Alert relay via cellular
6. Recording delivery to companion app

### IP67 Tests (Applicable Nodes)

1 m water depth, 30 minutes. Post-submersion: powers on, all sensors functional, rails within ±2%.

---

## Component Selection Philosophy

### No Pre-Constrained Specifications

Component lists in schematic READMEs are **recommendations and tested variants**, not locked specifications. Researchers may substitute:

- Different MCU families (ARM, RISC-V)
- Different sensor models with similar capabilities
- Different battery capacities
- Different radio modules (within regulatory compliance)

### Requirements for Substitutions

- Same interface type (SPI, I2C, MIPI, etc.)
- Power within PCB design limits
- Pin-compatible or minor rework required
- Regulatory compliance maintained

---

## Configuration Files

Hardware configuration exposed in `sentinel-wear.toml`:

```toml
[nodes]
pendant.type = "standard"           # "standard" | "360_curved" | "medallion" | "tactical"
pendant.camera = false
pendant.uwb = false
pendant.battery_mah = 250

bracelets.sensor_set = "standard"    # "minimal" | "standard" | "extended"
bracelets.camera = false
bracelets.uwb = false

belt.variant = "linux_som"           # "mcu" | "linux_som" | "high_perf" | "esp32" | "hotswap"
belt.cellular = true
belt.cellular_module = "ec25"        # "ec21" | "ec25" | "rm502q" | "eg912y"
belt.battery_mah = 5000

anklets.sensor_set = "standard"
anklets.uwb = true                   # Recommended for gait sync

eyewear.variant = "clip_on"          # "clip_on" | "frame"
eyewear.sensors = "event_only"       # "event_only" | "event_camera"

[ban]
primary = "ble"                      # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = true
uwb_role = "timing_and_ranging"      # "timing_only" | "timing_and_ranging" | "full_bandwidth"

[connectivity]
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
alert_via_cellular = true
```

---

## Legal and Compliance

### RF Compliance

- All BLE modules: FCC Part 15.247, CE RED Article 3.2
- All UWB modules: FCC Part 15.250, CE RED
- Cellular modules: Carrier certification required per module

### Electrical Safety

- UL 60950-1 / IEC 62368-1
- Battery: UL 1642 / IEC 62133

### EMC

- FCC Part 15 Subpart B (unintentional radiators)
- CISPR 32 Class B (residential)

### Export Control

See `legal/export_control_posture.md` for complete analysis. Summary:
- mmWave radar (60 GHz): Generally not export-controlled
- UWB: Generally not export-controlled in most jurisdictions
- Standard electronics: Standard export classifications apply

---

## References

- `docs/theory/ban_bandwidth_budget.md` — BLE/UWB allocation strategies
- `docs/theory/power_management.md` — Battery and power optimization
- `docs/theory/thermal_management.md` — Thermal design for wearables
- `docs/theory/rf_coexistence.md` — RF interference mitigation
- `firmware/firmware.md` — Firmware architecture per node type
- `mechanical/mechanical_spec.md` — Enclosure designs
- `legal/compliance.md` — Regulatory compliance details

---

## Contributing

When submitting hardware designs:

1. Confirm sensing-only scope (no effectors)
2. Confirm regulatory compliance (RF, electrical, EMC)
3. Provide complete BOM with supplier part numbers
4. Include test procedure for validation
5. Document any configuration options supported

PRs will be reviewed for scope compliance, technical correctness, and documentation completeness.

---

**License:** CERN-OHL-S v2 for hardware designs.
