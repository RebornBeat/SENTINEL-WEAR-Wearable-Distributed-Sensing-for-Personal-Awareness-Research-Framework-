Here is the fully updated, corrected, and expanded `README.md` for **SENTINEL-WEAR**. This version integrates the **Camera Identification Layer** as an opt-in, privacy-centric feature distinct from the primary sensing mesh, ensuring no omissions from previous iterations.

---

# SENTINEL-WEAR — Wearable Distributed Sensing for Personal Awareness (Research Framework)

**An open research framework for wearable, jewelry-form-factor distributed sensors that build a 360° body-coordinate awareness field around the wearer.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Sensing-Only](https://img.shields.io/badge/Effectors-Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

SENTINEL-WEAR is a **research and education** project that studies the architecture and feasibility of wearable distributed sensing in jewelry form factors — necklaces, bracelets, anklets, belt-pieces, eyewear — that together build a body-frame awareness field around the wearer. The project is motivated by the observation that personal protective adornments (literal or symbolic) are present in nearly every culture in human history, and that with modern miniaturized sensors, the *sensing* dimension of personal protection is now genuinely feasible in those form factors.

**Strictly limited to:**

- Sensor architecture (mmWave radar, IMU, low-power LiDAR, acoustic, environmental).
- Distributed perception in body-frame coordinates.
- Tracking and threat-classification algorithms (using PentaTrack).
- Alerting, haptic, and information-display research.
- Form-factor and human-factors research (comfort, weight, battery, charging, durability, water resistance).
- **Opt-in Visual Identification:** Low-resolution camera integration for specific nodes (e.g., Necklace) used strictly for object/face classification, not for primary motion tracking.
- **Feasibility Research:** Theoretical analysis of detection latency for high-velocity objects and survey of adjacent materials science domains.

**Explicitly out of scope:**

- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any mechanism designed to deflect, intercept, capture, or otherwise act physically against incoming objects, including but not limited to bullets, blades, or projectiles.
- Any "active protection system" actuator design at any scale.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

The reasoning is straightforward: physical interception at body scale is (a) currently dominated by passive armor, which is mature and well-served by existing products; (b) a regulated category in nearly every jurisdiction; and (c) genuinely dangerous to research informally. SENTINEL-WEAR's contribution is upstream of any actuation: *if you cannot perceive a threat, no effector matters.* If perception is solved well in a wearable form factor, downstream actuation becomes a regulated engineering problem for qualified parties — outside this repository.

See `docs/theory/future_research.md` for the full research-domain map across all tiers, and `legal/research_ethics.md` for the project's ethics posture.

---

## Innovation Landscape

The wearable sensing market is dominated by single-node devices (smartwatches, fitness trackers) that treat the body as a point and answer questions about that point — heart rate at the wrist, steps via wrist accelerometer, sleep via wrist motion. SENTINEL-WEAR sits in an under-published research space: distributed sensing across multiple body-worn nodes that together treat the body as a *volume* with surrounding context.

**Architectural focus areas:**

1.  **Distributed Body-Frame Fusion.** Multiple nodes (neck, wrists, waist, ankles, optionally eyewear) cooperate to maintain a unified body-frame coordinate system. Each node's local detections are transformed into the shared frame and fused. Published research on multi-IMU body-pose estimation provides the technical substrate; the application to jewelry-form-factor distributed perception is the open research question this project addresses.

2.  **Body-Frame Drift Correction.** The wearer is in motion — walking, turning, gesturing. A naive perception system reports the world rotating when the wearer turns. Body-frame drift correction uses the multi-IMU stack to subtract wearer motion from sensor observations, so a "threat approaching from behind" alert reflects an actual approach and not an artifact of the wearer turning around. See `docs/theory/body_coordinate_fusion.md`.

3.  **Sensing-Physics Research at Body-Frame Timescales.** A separate research track studies the physics constraints of detecting fast-moving objects within body-frame engagement distances. This is upstream-of-actuation work — characterizing what sensor architectures *can* and *cannot* detect at the relevant timescales — and is the substrate for any future research by qualified parties on the downstream questions. See `docs/theory/extreme_velocity_sensing.md`.

4.  **Separation of Sensing and Identification.** Unlike camera-based security systems, SENTINEL-WEAR decouples **Sensing** (detecting a presence/movement via radar/LiDAR) from **Identification** (classifying *who* or *what* via camera). This preserves privacy by ensuring cameras are only activated when necessary and only for classification, not continuous surveillance.

These are research focuses, not patent claims. The repository is MIT-licensed and the maintainers do not pursue patent filings on the architecture.

---

## What the Project Actually Builds

A **distributed wearable sensor mesh** worn on the body that continuously maintains a body-frame 3D awareness field. The wearer becomes the origin of their own coordinate system; sensors at multiple body locations cover overlapping hemispheres around the body.

```
                       ┌───────────────┐
                       │ Eyewear Node  │  forward hemisphere, head-stabilized
                       └───────┬───────┘
                               │
                       ┌───────┴───────┐
                       │ Neck/Pendant  │  upper-360° azimuth, elevation up
                       │     Node      │  (Optional: Visual Identification)
                       └───┬───────┬───┘
                           │       │
                  ┌────────┘       └────────┐
                  │                         │
            ┌─────┴─────┐             ┌─────┴─────┐
            │ Bracelet  │             │ Bracelet  │   forearm-frame coverage,
            │   Left    │             │   Right   │   gesture + arm-direction
            └───────────┘             └───────────┘
                          ┌────────────┐
                          │ Belt Node  │   torso-frame, downward coverage,
                          │            │   primary compute / battery
                          └─────┬──────┘
                                │
                  ┌─────────────┴─────────────┐
                  │                           │
            ┌─────┴─────┐               ┌─────┴─────┐
            │  Anklet   │               │  Anklet   │   ground-plane coverage,
            │   Left    │               │   Right   │   gait, fall detection
            └───────────┘               └───────────┘
```

Each node carries some subset of:

- **mmWave radar** — presence/motion of objects approaching the body envelope; functions in darkness, through obscuration, and in adverse weather. **(Primary Sensing Modality)**
- **IMU** — orientation of that node in the body frame; the substrate for body-frame drift correction.
- **Microphone array** — acoustic localization, direction-of-arrival of sudden sounds (glass break, verbal aggression, vehicle approach).
- **Short-range LiDAR / ToF** — close-range geometry where radar resolution is insufficient.
- **Environmental sensors** — temperature, gas, air quality.
- **Haptic actuator** — silent directional alerts to the wearer.
- **Low-resolution Camera (Optional/Opt-in)** — **Identification Layer.** Not used for tracking movement. Used strictly to classify detected objects (e.g., "Person known: John" or "Object held: Knife" vs "Stick"). This module is typically located in the **Necklace/Pendant Node** due to its wider, stable field of view (POV) and is physically or logically disabled by default to ensure privacy.

The **belt node** is the primary compute and battery — easiest to make heavier without compromising comfort. Other nodes are slim and beam metadata to the belt over a low-power body-area-network radio.

---

## Body-Frame Tracking with PentaTrack

The fusion engine uses [PentaTrack](https://github.com/RebornBeat/PentaTrack) but with the coordinate origin attached to the wearer's *torso*, not the world. Every node's IMU continuously reports its orientation; the fusion layer transforms each node's local detections into a unified body-frame, applies drift correction to subtract wearer motion, and produces a tracked field of objects in body coordinates:

- "Person approaching from behind, 4 m, closing at 1.5 m/s."
- "Vehicle 8 m to the left, parallel motion, 30 km/h."
- "Sudden acoustic event 2 m forward-up, glass-break signature, 0.91 confidence."
- "Wearer gait deviation: 60% probability of stumble in the next 800 ms."

PentaTrack's object-type awareness, drift profiles, and anomaly detection are repurposed: the "object types" become "human walking," "human running," "vehicle," "animal," "thrown object" — each with drift profiles tuned for that class's plausible motion in body-frame coordinates.

---

## Research Domain: Extreme Velocity Sensing

A specific research track of this project studies the physics of detecting high-velocity objects at body-frame engagement distances. This is sensing-physics research — characterizing reaction-time budgets, the failure modes of standard sensor modalities at relevant timescales, and the architecture of sensor stacks that can operate within those budgets.

### The Physics Problem
Standard sensing technologies fail at ballistic speeds.
*   **LiDAR:** Too slow. Scanning latency creates blind spots between scans. A bullet moves 40+ meters between typical LiDAR scans.
*   **Acoustic:** Too slow. Sound travels at ~343 m/s. A supersonic bullet arrives *before* the sound of the muzzle blast. Acoustic sensing is forensic, not preventive.

### The Proposed Sensor Architecture
The track produces a specification for the only viable sensor fusion stack:
- **Doppler Radar:** Continuous Wave (CW) radar has zero scanning latency and detects velocity shifts instantaneously.
- **Event-Based Vision:** Neuromorphic cameras react in microseconds, capturing the trajectory without the bandwidth overhead of standard video.

This research documents the **Reaction Time Budget**: for a 3-meter engagement radar-style display.
-   **Identification Tags (Opt-in).** If the optional Visual Identification Layer (Necklace Node) is enabled, the system attaches classification tags to tracked objects (e.g., "Known Contact: Sarah" or "Object: Knife"). This allows the wearer to distinguish between a known friend approaching and an unknown entity without needing to turn around.
-   **Logging** for personal review.
-   **Optional emergency contact** — manually triggered, never automatic — that sends a position to a designated contact.

What SENTINEL-WEAR explicitly does *not* do: trigger any physical mechanism, deploy any object, release any substance, or take any kinetic action of any kind.

---

## Repository Layout

```
sentinel-wear/
├── docs/
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
│   │   └── why_sensing_only.md
│   ├── api/
│   └── assets/
├── hardware/
│   ├── schematic/
│   │   ├── pendant_node/        # Includes optional camera module
│   │   ├── bracelet_node/
│   │   ├── belt_node/
│   │   ├── anklet_node/
│   │   └── eyewear_node/
│   ├── gerbers/
│   ├── bom/
│   ├── assembly/
│   ├── datasheets/
│   ├── testing/
│   ├── hardware_config.md
│   └── README.md
├── firmware/
│   ├── src/
│   │   ├── drivers/
│   │   ├── logic/
│   │   ├── ban_protocol/
│   │   └── main.c
│   ├── bootloader/
│   ├── tools/
│   ├── platformio.ini
│   ├── firmware.md
│   └── README.md
├── software/
│   ├── belt_controller/
│   ├── mobile/
│   ├── desktop/
│   ├── cli/
│   ├── sdk/
│   ├── protocol/
│   │   └── protocol_spec.md
│   ├── software.md
│   └── README.md
├── mechanical/
│   ├── cad/
│   ├── stl/
│   ├── drawings/
│   ├── mechanical_spec.md
│   └── README.md
├── production/
├── legal/
│   ├── compliance.md
│   ├── medical_device_disclaimer.md
│   ├── research_ethics.md
│   ├── export_control_posture.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

---

## Roadmap

-   **Phase 1.** Belt-node-only bench prototype — full sensor stack, body-frame fusion of one node, IMU-driven drift correction.
-   **Phase 2.** Add bracelet + pendant — demonstrate multi-node body-frame fusion and cross-node drift correction.
-   **Phase 3.** Full mesh of all six nodes with BAN protocol. Integration of optional **Visual Identification Layer** on Necklace node (opt-in).
-   **Phase 4.** Comfort, durability, weight, water resistance, hypoallergenic-material, and human-factors study.
-   **Phase 5.** Public open-data release of body-frame trajectory dataset for the research community.
-   **Parallel Track (sensing physics).** Reaction-time-budget analysis and sensor-architecture characterization for fast-moving-object detection at body-frame engagement distances.

There is no Phase N for actuation. By design.

---

## Disclaimer

> SENTINEL-WEAR is a research and education project. It is not a medical device, not a personal protective equipment certification, not a self-defense product, not a substitute for situational awareness, and not a substitute for any law-enforcement, medical, or emergency service. The maintainers make no warranty of fitness for any safety-critical use. Use at your own risk for educational purposes only.

---

## License

MIT for code. CERN-OHL-S v2 for hardware. CC BY 4.0 for documentation.
