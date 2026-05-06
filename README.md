# SENTINEL-WEAR — Distributed Body-Frame Wearable Sensing Mesh

**An open research framework for wearable, jewelry-form-factor distributed sensors that build a 360° body-coordinate awareness field around the wearer. Full environmental capture. Full world reconstruction. Full user control.**

[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](#license)
[![License: CERN OHL](https://img.shields.io/badge/Hardware-CERN%20OHL--S%20v2-green.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Sensing-Only Effectors](https://img.shields.io/badge/Effectors-Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

SENTINEL-WEAR is a **research and education** project studying the architecture and feasibility of wearable distributed sensing in jewelry form factors — necklaces, bracelets, anklets, belt-pieces, eyewear — that together build a body-frame awareness field around the wearer. The project is motivated by the observation that personal protective adornments (literal or symbolic) are present across nearly every culture in human history, and that with modern miniaturized sensors, the *sensing* dimension of personal protection is now genuinely feasible in those form factors.

**Strictly limited to:**
- Sensor architecture (mmWave radar, IMU, low-power LiDAR, acoustic, environmental, event cameras, conventional cameras, 360° imaging).
- Distributed perception in body-frame coordinates.
- Tracking and threat-classification algorithms (PentaTrack).
- Alerting, haptic, and information-display research.
- Form-factor and human-factors research (comfort, weight, battery, charging, durability, water resistance, hypoallergenic materials).
- Full world reconstruction — both sparse probabilistic models and dense SLAM-based 3D maps.
- Opt-in visual identification for any node class via camera integration.
- Feasibility research on detection physics at body-frame timescales.
- Storage, streaming, and legal evidence capture of all sensor data.

**Explicitly out of scope:**
- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any mechanism designed to deflect, intercept, capture, or otherwise act physically against incoming objects.
- Any active protection system actuator design at any scale.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

---

## What the Project Actually Builds

A **distributed wearable sensor mesh** worn on the body that continuously maintains a body-frame 3D awareness field. The wearer becomes the origin of their own coordinate system; sensors at multiple body locations cover overlapping hemispheres around the body. The system supports two parallel world-model modes simultaneously:

**Mode A — Sparse Probabilistic World Model:** PentaTrack-based, low-power, event-driven. Fast, efficient. Answers: "Something is here, moving this way, at this speed."

**Mode B — Dense SLAM World Map:** Full 3D reconstruction from LiDAR + camera arrays. Geometry-complete. Answers: "This is what the environment looks like, with objects identified and tracked." Requires higher compute (belt node Linux SoM variant or companion app offload).

Both modes run simultaneously when hardware supports it. Mode A always runs. Mode B activates based on hardware configuration and user preference.

┌─────────▼────────────┐  ┌──────────▼──────────────┐  ┌─────────▼──────────┐
│  Pendant / Necklace  │  │  Pendant / Necklace      │  │  Pendant Standard  │
│  Node — Standard     │  │  Node — 360° Curved      │  │  Node — Medallion  │
│  Flat pendant form   │  │  Curved flexible PCB     │  │  Premium housing   │
│  mmWave+IMU+Acoustic │  │  8-camera 360°+mmWave    │  │  mmWave+IMU+Acous  │
│  + opt-in cam        │  │  +IMU+Acoustic+Radar     │  │  +opt-in full cam  │
└─────────────────────┘  └─────────────────────────┘  └────────────────────┘
│
┌─────────────┴──────────────┐
│                            │
┌────────────▼────────┐     ┌────────────▼────────┐
│  Bracelet Left      │     │  Bracelet Right     │
│  mmWave+IMU+Haptic  │     │  mmWave+IMU+Haptic  │
│  + opt-in cam       │     │  + opt-in cam       │
└─────────────────────┘     └────────────────────┘
│
┌──────────▼──────────┐
│      Belt Node       │
│  PRIMARY HUB         │
│  Compute+Battery     │
│  IMU(Torso Ref)      │
│  mmWave(down)+Env    │
│  App Server + API    │
│  Recording Manager   │
│  SLAM Processor      │
└──────────┬───────────┘
│
┌─────────────┴──────────────┐
│                            │
┌────────────▼────────┐     ┌────────────▼────────┐
│  Anklet Left        │     │  Anklet Right       │
│  ToF/LiDAR+IMU+     │     │  ToF/LiDAR+IMU+     │
│  Haptic+opt cam     │     │  Haptic+opt cam     │
└─────────────────────┘     └────────────────────┘

---

## Innovation Landscape

### Wearable Distributed Sensing Gap

The wearable sensing market is dominated by single-node devices (smartwatches, fitness trackers) that treat the body as a point. SENTINEL-WEAR sits in an under-published research space: distributed sensing across multiple body-worn nodes that together treat the body as a *volume* with surrounding context.

**Architectural focus areas:**

1. **Distributed Body-Frame Fusion.** Multiple nodes (neck, wrists, waist, ankles, optionally eyewear) cooperate to maintain a unified body-frame coordinate system. Each node's local detections are transformed into the shared frame and fused. PentaTrack provides the predictive center-field output.

2. **Body-Frame Drift Correction.** The wearer is in motion — walking, turning, gesturing. The multi-IMU stack subtracts wearer motion from sensor observations so that "threat approaching from behind" reflects actual approach, not an artifact of the wearer turning. See `docs/theory/body_coordinate_fusion.md`.

3. **Dual World Model Architecture.** Both sparse probabilistic tracking and dense SLAM reconstruction are first-class outputs. Neither is a compromise. Hardware configuration determines which is active.

4. **360° Curved Pendant Design.** A custom flexible or rigid-flex PCB pendant with cameras distributed at angular positions around the necklace circumference, producing continuous 360° visual coverage from the chest position.

5. **Sensing-Physics Research at Body-Frame Timescales.** Characterizes what sensor architectures can and cannot detect at the relevant timescales for fast-moving objects. Upstream-of-actuation sensing physics research.

6. **Full Data Ownership.** Users own all data captured by their system. Raw recordings, compressed clips, metadata — all configurable. All storage local by default. Streaming to app on user request. Legal export with cryptographic integrity chain.

---

## Node Architecture — All Variants

### Pendant / Necklace Node

The highest-information-value node. Three variants:

**Variant A — Standard Flat Pendant**
- Flat PCB in a medallion-style enclosure
- mmWave radar (forward + lateral hemisphere)
- IMU (body-frame orientation)
- Microphone array (acoustic DOA + material classification)
- Optional camera module (single forward-facing, opt-in)
- Optional environmental sensors
- 150–300 mAh battery
- ~30–60 g total

**Variant B — 360° Curved Pendant / Necklace**
- Custom curved flexible PCB or rigid-flex following the necklace arc
- 4–8 camera modules distributed around the circumference at angular positions
- Example: 8 cameras at 0°, 45°, 90°, 135°, 180°, 225°, 270°, 315° (full 360°)
- Image stitching on-pendant (dedicated ISP) or offloaded to belt node
- mmWave radar arrays at multiple angular positions for 360° radar coverage
- Distributed IMU array for orientation at each arc segment
- Distributed microphone array matching camera arc
- Sub-pendant accessory form factor — hangs naturally, fits into jewelry aesthetic
- Custom PCB production required (flexible substrate, custom component placement)
- Belt node handles full 360° video stitching, SLAM integration, and recording
- This variant enables true 360° continuous world capture from chest-level position
- 400–800 mAh battery (distributed in pendant arc or separate battery module)

**Variant C — Medallion (Premium)**
- Larger pendant with thicker profile
- Houses higher-compute local processor
- Higher-resolution cameras (single or dual)
- More microphones (6-element tetrahedral array)
- Integrated LiDAR / structured light
- High-capacity battery (up to 1000 mAh)
- ~70–120 g total

### Bracelet Node (×2)

Forearm-hemisphere sensing and directional haptic output.

**Variant A — Minimal**
- Acconeer XR112 radar + BMI270 IMU + LRA haptic
- nRF5340 MCU
- Ultra-slim band (< 3 mm z-profile)

**Variant B — Standard**
- mmWave radar + IMU + LRA haptic
- Optional short-range ToF
- 200–400 mAh

**Variant C — Extended**
- mmWave radar + IMU + LRA haptic
- Single outward-facing camera (opt-in, user-configured)
- Larger battery
- Useful for gesture recognition and arm-direction visual awareness

### Belt Node

Primary compute, battery hub, torso reference, app server.

**Variant A — MCU (Minimal)**
- STM32H7 class
- Local embedded RTOS
- Runs `sentinel-belt-controller` Rust binary
- ~2000 mAh

**Variant B — Linux SoM (Full)**
- Raspberry Pi CM4 / NXP i.MX 8M Plus
- Full Linux stack
- SLAM processing, companion app server, recording management
- 5000–7000 mAh battery bank

**Variant C — High-Performance**
- Qualcomm SA8155P or equivalent
- GPU-class inference for dense SLAM
- Can run neural 360° stitching from pendant cameras locally
- Enables full offline dense world reconstruction without companion app

### Anklet Node (×2)

Ground-plane sensing, gait analysis, lower-hemisphere coverage.

**Variant A — Minimal**
- VL53L5CX ToF + BMI270 IMU + LRA haptic
- 300 mAh

**Variant B — Extended**
- Short-range LiDAR + mmWave + IMU + haptic
- Optional small downward-facing camera (ground-level obstacle detection)
- 500 mAh

### Eyewear Node (Optional)

Head-stabilized forward-hemisphere sensing.

**Variant A — Event-Only**
- Event camera + IMU
- Clip-on form factor

**Variant B — Event + Camera**
- Event camera + conventional camera + IMU
- Better for SLAM integration (provides forward visual odometry)

**Variant C — Full Array**
- Front event camera + front conventional camera + side cameras
- Head-mounted 180°+ coverage
- Suitable for full SLAM integration as visual odometry anchor

---

## World Model Architecture

SENTINEL-WEAR supports two simultaneous world model modes. Both are always available; hardware configuration determines the fidelity of each.

### Mode A — Sparse Probabilistic Body-Centric World Model

**Technology stack:** OMNI-SENSE → PentaTrack → Body-frame stabilizer

**What it produces:**

TrackedEntity {
id
position (x, y, z)     // in body frame
velocity (vx, vy, vz)  // in body frame
prediction_centers     // PentaTrack center field
confidence
classification_hint    // "human", "vehicle", "animal"
anomaly_flags
}

**Visualization:** Radar-style body-centric display. Detected entities shown as blobs with velocity vectors and PentaTrack prediction arcs. Confidence shown as opacity/size. Alert zones shown as colored rings.

**Use cases:** All alert generation, gait analysis, directional haptic routing, low-latency real-time awareness.

**Power:** Runs on all hardware configurations. Always active.

### Mode B — Dense SLAM World Map

**Technology stack:** LiDAR + cameras → SLAM → 3D mesh → Object recognition → Tracked entities overlay

**What it produces:**

WorldMap {
dense_3d_mesh         // full geometry of environment
object_instances      // detected and classified objects
tracked_entities      // same as Mode A but geometry-anchored
camera_feeds          // per-camera streams with entity overlays
stitched_panorama     // 360° view from pendant cameras (if 360° variant)
}

**Visualization:** Full 3D world reconstruction. Users can fly-through the captured environment, review recordings in 3D context, overlay tracked entities on the geometry.

**Use cases:** Full environment review, legal evidence, SLAM-based odometry for body-frame enhancement, 360° panoramic recording.

**Power:** Requires belt node Linux SoM variant or companion app compute offload.

### Dual Mode Simultaneous Operation

When belt node is Linux SoM class, both modes run simultaneously:
- Mode A is the real-time alert and tracking backbone (always active, low latency)
- Mode B builds in background for rich review, replay, and export
- Mode A benefits from Mode B: object classifications from SLAM improve PentaTrack drift profiles
- Mode B benefits from Mode A: PentaTrack predictions annotate the dense map with motion intent

---

## 360° Curved Pendant — Technical Architecture

The 360° curved pendant is a signature variant of SENTINEL-WEAR. It transforms the pendant node from a directional sensor to a continuous 360° capture platform.

### Flexible PCB Architecture

The pendant hangs from a necklace chain. A curved flexible PCB (or rigid-flex hybrid) follows the arc of the pendant face. Camera modules are placed at regular angular intervals around the circumference.

**Physical layout (example — 8-camera variant):**

Top of pendant (worn at chest)
┌────────────────────────────────┐
│ 315°  0°  45°                  │  ← Front face cameras (3 cameras)
│                                │
│ 270°        90°                │  ← Side cameras (2 cameras)
│                                │
│ 225°  180°  135°               │  ← Rear / back cameras (3 cameras)
└────────────────────────────────┘
     Bottom of pendant

**Camera spacing and overlap:**
- 8 cameras at 45° intervals: each camera needs ~60° FoV for full overlap
- 6 cameras at 60° intervals: each camera needs ~80° FoV for full overlap
- 4 cameras at 90° intervals: each camera needs ~100° FoV (wide-angle required)

**PCB construction options:**
- **Rigid-flex:** Rigid PCB segments joined by flexible bridges. Best for component density and reliability.
- **Fully flexible:** Single-layer or multi-layer flexible PCB. Better conformability. More fragile.
- **Modular array:** Individual camera modules connected by a flexible ribbon cable. Allows independent replacement.

### Image Processing Pipeline

**On-pendant ISP (if compute allows):**
- Each camera produces a raw frame
- On-pendant ISP chip stitches to equirectangular 360° frame
- Compressed stream transmitted to belt node over high-bandwidth link (USB 3.x or custom RF)

**Belt-node processing:**
- Raw camera streams from all cameras transmitted individually
- Belt node Linux SoM handles stitching (GPU-accelerated if available)
- Real-time 360° video recording to SD card
- Companion app receives stitched stream for live 360° viewing

**Companion app processing:**
- For lowest pendant complexity: raw frames from all cameras transmitted to app
- App handles stitching in software
- Enables highest-quality output at the cost of bandwidth

### SLAM Integration

The 360° pendant with SLAM is extremely powerful:
- 360° visual input at chest level
- Combined with anklet ToF, LiDAR node data, and belt IMU
- Produces a continuously-updated 3D map anchored to the body
- As the wearer moves, new areas of the environment are mapped
- The SLAM map becomes a persistent 3D record of everywhere the wearer has been

This is equivalent capability to a stationary 360° security camera array — but mobile, wearable, and body-anchored.

---

## Sensor Stack

┌──────────────────────────────────────────────────────────────────────────────┐
│                        SENTINEL-WEAR Sensing Stack                           │
├──────────────────────────────────────────────────────────────────────────────┤
│  Always-On Layer (Low Power — All Nodes)                                     │
│  mmWave radar (presence, motion, micro-Doppler activity signatures)           │
│  IMU (body-frame orientation, gait, gesture)                                  │
│  PIR (optional — binary presence at specific nodes)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│  High-Fidelity Layer (Event-Triggered or Continuous)                          │
│  Solid-state LiDAR / ToF (geometry, ground clearance, obstacle detection)     │
│  Event-based camera (microsecond-latency fast object detection)               │
│  Microphone array (acoustic DOA, material classification, event detection)    │
│  Environmental sensors (temperature, humidity, VOC, air quality)              │
├──────────────────────────────────────────────────────────────────────────────┤
│  Visual Capture Layer (User-Configured — Any Node)                            │
│  Conventional cameras at any angular position on any node                     │
│  360° curved pendant multi-camera array (signature variant)                   │
│  Full video recording (raw, compressed, continuous, or on-trigger)            │
│  Live streaming to companion app                                               │
│  SLAM-ready visual odometry output                                             │
└──────────────────────────────────────────────────────────────────────────────┘

---

## Software Architecture

### Crate Structure

sentinel-wear/
├── crates/
│   ├── sentinel-core/              # Shared types, config, node descriptors, events
│   ├── sentinel-perception/        # Per-node sensor pipelines (OMNI-SENSE based)
│   ├── sentinel-body-frame/        # Multi-IMU body-frame fusion, stabilization, calibration
│   ├── sentinel-fusion/            # Multi-node detection fusion (CI + JPDA)
│   ├── sentinel-tracking/          # PentaTrack bridge, body-space prediction
│   ├── sentinel-extreme-velocity/  # High-speed / sprint / collision prediction
│   ├── sentinel-slam/              # SLAM pipeline, 360° stitching, dense world model
│   ├── sentinel-ban-protocol/      # BAN protocol (BLE 5.x / UWB)
│   ├── sentinel-alerts/            # Alert classification, haptic encoding, routing
│   ├── sentinel-storage/           # Recording management, retention, export
│   ├── sentinel-api/               # Companion app REST/WebSocket/media API server
│   └── sentinel-belt-controller/   # Main binary: runs on belt node
├── firmware/                       # no_std embedded firmware per node type
├── hardware/                       # PCB schematics, BOMs, test jig
│   ├── schematic/
│   │   ├── pendant_node/           # All pendant variants including 360°
│   │   ├── bracelet_node/
│   │   ├── belt_node/
│   │   ├── anklet_node/
│   │   └── eyewear_node/
│   ├── testing/
│   │   └── test_jig_pcb/
│   └── hardware_config.md
├── apps/                           # Companion app architecture and API spec
├── docs/                           # Theory, guides, API reference
│   ├── guides/
│   │   ├── getting_started.md
│   │   ├── body_frame_calibration.md
│   │   └── alert_modalities.md
│   ├── theory/
│   │   ├── future_research.md
│   │   ├── body_coordinate_fusion.md
│   │   ├── drift_profiles_at_body_scale.md
│   │   ├── form_factor_human_factors.md
│   │   ├── extreme_velocity_sensing.md
│   │   ├── passive_materials_research.md
│   │   └── why_no_actuation.md
│   ├── api/
│   │   └── api_spec.md
│   └── protocol/
│       └── protocol_spec.md
├── scenarios/                      # Simulation scenarios
├── mechanical/                     # CAD, STL, enclosure designs
├── production/                     # Manufacturing SOPs, QC
├── legal/                          # Compliance, ethics, export control
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md

### Dependency Chain

OMNI-SENSE (sensor abstraction, physics, fusion)
↓
sentinel-core (shared types, events, config)
↓
sentinel-perception (node sensor pipelines)
sentinel-body-frame (IMU fusion, coordinate transforms, calibration)
↓
sentinel-fusion (multi-node CI + JPDA)
sentinel-slam (360° stitching, SLAM, dense world model)
↓
sentinel-tracking (PentaTrack bridge, world model integration)
sentinel-extreme-velocity (high-speed prediction)
sentinel-storage (recording management)
↓
sentinel-alerts (haptic routing, notification)
sentinel-api (companion app server)
↓
sentinel-belt-controller (main binary)

## Time Synchronization

Distributed wearable sensing requires precise clock alignment across all body nodes.

**Belt node as time master:** Broadcasts synchronized timestamps to all nodes at each BLE connection event interval.

**Per-node offset estimation:** Each node runs `omni-sense-time::ClockOffsetEstimator` (PTP-style four-timestamp exchange) tracking local clock drift with continuous correction.

**UWB upgrade path:** Qorvo DW3000-class UWB available on all nodes for sub-nanosecond ranging and time alignment when required for SLAM accuracy.

**360° pendant sync:** The 360° curved pendant requires tight synchronization across all camera modules for clean stitching. Camera sync lines within the pendant PCB provide hardware-level sync; the belt node time master provides cross-pendant timing reference.

---

## Body-Frame Coordinate System

The belt node's IMU defines the body-frame origin:
- **+X:** Wearer's right
- **+Y:** Wearer's forward
- **+Z:** Up (gravity direction inverted)

All node detections are expressed in this stabilized frame via `omni-sense-frames::BodyFrameStabilizer`. Stabilization mode (World, Translational, None) configurable per application.

Walk-through calibration (`sentinel-body-frame::WalkThroughCalibrator`) establishes geometric offsets of each node relative to the torso origin. The 360° pendant calibration includes angular offset of each camera module relative to the pendant center.

---

## Privacy Model — User Configured

SENTINEL-WEAR has no system-imposed data restrictions. All data handling is user-configured:

**Privacy controls available (all user-selected):**
- Physical power switch on any camera module (user cuts power when desired)
- Software disable mode (camera driver disabled in firmware config)
- Activity-based recording (record only on detection trigger)
- Schedule-based recording (e.g., off during sleep)
- No recording (metadata-only mode)

**No mandatory kill switch requirement.** The architecture supports hardware switches for users who want them. It also supports software-only privacy controls for users who prefer them. Both are valid configurations. The system does not mandate either approach.

**Data handling options (all user-configured in `sentinel-wear.toml`):**
- Metadata-only transmission (classification results only)
- Raw local storage (SD card, internal flash)
- App streaming (live or on-demand)
- Full continuous recording
- Tiered: metadata by default, raw on alert
- Any combination of the above

---

## Data Storage and Companion App

### Storage Options

All node classes support local storage:
- microSD card (recommended — removable for physical access, up to 2 TB)
- Internal flash (non-removable, smaller capacity)
- Network storage via belt node Wi-Fi

### Companion App

Mobile (iOS/Android) and desktop (Windows/macOS/Linux) apps connect to the belt node's embedded server.

**Core features:**
- Live sensor display (both sparse world model and dense SLAM view)
- 360° panoramic live view from curved pendant cameras
- Full recordings library with playback, search, timeline scrubbing
- 3D replay with time-scrubbing through the SLAM world model
- Gait analytics, event history, anomaly logs
- System configuration
- Legal export with cryptographic integrity chain

See `apps/README.md` for full API and architecture specification.

---

## Companion App World Model Views

### View 1 — Radar / Event Display (always available)

Body-centric radar-style display:
- Wearer at center
- Detected entities as blobs with velocity vectors
- PentaTrack prediction arcs
- Alert zones as colored rings
- Entity classification labels
- Confidence as opacity
- Gait event annotations

### View 2 — 360° Live Panorama (360° pendant variant)

Equirectangular or spherical 360° live view from pendant cameras:
- All camera streams stitched in real-time
- Entity detections overlaid as bounding boxes
- Works in playback for recordings review
- Supports VR headset viewing

### View 3 — Dense SLAM 3D World Map (Linux SoM belt variant)

Interactive 3D world model:
- Full geometry of every visited environment
- Objects classified and labeled
- Tracked entity trajectories overlaid
- Time-scrubbing to replay any moment in 3D context
- Useful for forensic review, legal preparation, security analysis

---

## Output Modalities

What SENTINEL-WEAR does with what it perceives:

- **Haptic alerts:** Directional buzz on the appropriate node — "approach from right rear" buzzes the right-rear node.
- **Audio alerts:** Bone conduction or earpiece (optional).
- **Companion-app overlay:** Live world model on phone, tablet, or desktop.
- **360° live view:** From curved pendant cameras to any connected display.
- **Recordings:** All data stored locally per user configuration. Available for review, export, legal use.
- **Legal evidence export:** Cryptographic hash chain, timestamp, device ID embedded in exported files.
- **Emergency contact:** Manually triggered (never automatic) location + clip share to designated contact.

---

## Gait Analysis

The anklet IMUs are the primary input for gait analysis:
- Step frequency (cadence)
- Step regularity (gait coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right comparison)
- Pre-stumble signature detection

These feed PentaTrack's `WearerSelfMotion` anomaly detection and generate predictive stumble alerts before falls occur.

---

## Extreme Velocity Sensing Research Track

A research track studies the physics of detecting fast-moving objects at body-frame engagement distances.

**The Physics Problem:**
- Standard LiDAR: scanning latency creates blind spots. Fast-moving objects traverse meters between scans.
- Acoustic: sound travels 343 m/s. Supersonic objects arrive before sound.

**The Viable Sensor Stack:**
- Doppler radar (CW): zero scanning latency, instantaneous velocity shift detection.
- Event-based vision: microsecond reaction time.

**Reaction Time Budget:** At 3-meter engagement distance, 2.5 ms (rifle-velocity) to 8.5 ms (handgun-velocity) from object entry to detection and alert processing.

This research documents detection capabilities. It does not address physical interception — that is explicitly out of scope.

See `docs/theory/extreme_velocity_sensing.md` for the full document.

---

## Civilian Transfer Applications

- Fall detection and gait analysis for elderly or mobility-impaired individuals
- Sports performance analytics (stride, cadence, impact forces)
- Physical therapy monitoring and progress tracking
- Industrial worker safety (proximity alerts in factories and construction)
- Personal security for vulnerable populations (journalists, aid workers)
- Research platform for distributed wearable sensing architecture

---

## Firmware

Embedded firmware (`firmware/`) runs on `no_std` Rust targeting ARM Cortex-M (STM32, nRF5340, i.MX RT). Each node type has a dedicated binary. Shared logic (drivers, BAN protocol) in `firmware/src/lib.rs`.

See `firmware/firmware.md` for the full firmware architecture specification.

---

## Hardware

Reference PCB designs for all node variants:
- `hardware/schematic/pendant_node/` — Standard flat, 360° curved, medallion variants
- `hardware/schematic/bracelet_node/` — Minimal, standard, extended variants
- `hardware/schematic/belt_node/` — MCU, Linux SoM, high-performance variants
- `hardware/schematic/anklet_node/` — Minimal and extended variants
- `hardware/schematic/eyewear_node/` — Event-only, event+camera, full-array variants
- `hardware/testing/test_jig_pcb/` — Production test jig
- `hardware/hardware_config.md` — Interface standards, pin maps, power architecture

---

## Roadmap

- **Phase 1.** Belt-node-only bench prototype — full sensor stack, body-frame fusion of one node, IMU-driven drift correction.
- **Phase 2.** Add bracelet + pendant — demonstrate multi-node body-frame fusion and cross-node drift correction.
- **Phase 3.** Full mesh of all six nodes with BAN protocol. Walk-through calibration. Sparse world model complete.
- **Phase 4.** 360° curved pendant prototype. Camera array stitching. Dense SLAM integration.
- **Phase 5.** Comfort, durability, weight, water resistance, hypoallergenic-material, and human-factors study.
- **Phase 6.** Companion app — both world model views operational. Full recording and legal export.
- **Phase 7.** Public open-data release of body-frame trajectory dataset for the research community.
- **Parallel Track.** Extreme-velocity sensing physics characterization.

---

## Getting Started

```bash
git clone https://github.com/ungatedminds/sentinel-wear
cd sentinel-wear
cargo build --workspace
cargo test --workspace
cargo run -p sentinel-belt-controller -- --simulated --config scenarios/sim_basic.toml
cd firmware
cargo build --target thumbv7em-none-eabihf --bin pendant_node --release
```

---

## Disclaimer

SENTINEL-WEAR is a research and education project. It is not a medical device, not certified personal protective equipment, not a self-defense product, and not a substitute for any law-enforcement, medical, or emergency service. The maintainers make no warranty of fitness for any safety-critical use. Use at your own risk for educational purposes only.

---

## License

MIT for code. CERN-OHL-S v2 for hardware. CC BY 4.0 for documentation.
