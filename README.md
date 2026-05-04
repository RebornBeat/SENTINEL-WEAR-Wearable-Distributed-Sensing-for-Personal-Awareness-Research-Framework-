# SENTINEL-WEAR — Wearable Distributed Sensing for Personal Awareness (Research Framework)

**An open research framework for wearable, jewelry-form-factor distributed sensors that build a 360° body-coordinate awareness field around the wearer.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Sensing-Only](https://img.shields.io/badge/Effectors-Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

SENTINEL-WEAR is a **research and education** project that studies the architecture and feasibility of wearable distributed sensing in jewelry form factors — necklaces, bracelets, anklets, belt-pieces, eyewear — that together build a body-frame awareness field around the wearer. The project is motivated by the observation that personal protective adornments (literal or symbolic) are present across nearly every culture in human history, and that with modern miniaturized sensors, the *sensing* dimension of personal protection is now genuinely feasible in those form factors.

**Strictly limited to:**

- Sensor architecture (mmWave radar, IMU, low-power LiDAR, acoustic, environmental).
- Distributed perception in body-frame coordinates.
- Tracking and threat-classification algorithms (using PentaTrack).
- Alerting, haptic, and information-display research.
- Form-factor and human-factors research (comfort, weight, battery, charging, durability, water resistance).
- **Feasibility Research:** Theoretical analysis of detection latency for high-velocity objects (see `docs/theory/extreme_velocity_sensing.md`).

**Explicitly out of scope:**

- Any kinetic, projectile, energetic, pyrotechnic, or chemical effector.
- Any mechanism designed to deflect, intercept, capture, or otherwise act physically against incoming objects, including but not limited to bullets, blades, or projectiles.
- Any "active protection system" actuator design at any scale.
- Any device that could be misclassified as a weapon or restricted defensive device under any jurisdiction.

The reasoning is straightforward: physical interception at body scale is (a) currently dominated by passive armor, which is mature and well-served by existing products; (b) a regulated category in nearly every jurisdiction; and (c) genuinely dangerous to research informally. SENTINEL-WEAR's contribution is upstream of any actuation: *if you cannot perceive a threat, no effector matters.* If perception is solved well in a wearable form factor, downstream actuation becomes a regulated engineering problem for qualified parties — outside this repository.

---

## Innovation Landscape

Existing patents in the wearable space typically cover single-node sensing (wrist-based heart rate) or simple gesture recognition. SENTINEL-WEAR innovates by treating the body not as a point, but as a **volume**.

**Key Architectural Innovations:**
1.  **Distributed Body-Frame Fusion:** Unlike a smartwatch that tracks the wrist, SENTINEL-WEAR fuses data from neck, wrists, and ankles to maintain a stable "torso-relative" coordinate system.
2.  **Body-Frame Drift Correction:** A critical algorithm that separates the wearer's motion (turning, walking) from the world's motion. This ensures that a "threat approaching" alert is real, and not an artifact of the wearer simply turning their body.
3.  **Body-Area Phased Array:** By synchronizing mmWave radar nodes across the body, the system can conceptually operate as a distributed aperture, improving angular resolution beyond what a single jewelry-sized sensor can achieve.

---

## What the Project Actually Builds

A **distributed wearable sensor mesh** worn on the body that continuously maintains a body-frame 3D awareness field. The wearer becomes the origin of their own coordinate system; sensors at multiple body locations (neck, wrists, waist, ankles, optionally eyewear) cover overlapping hemispheres around the body.

```
                       ┌───────────────┐
                       │ Eyewear Node  │  forward hemisphere, head-stabilized
                       └───────┬───────┘
                               │
                       ┌───────┴───────┐
                       │ Neck/Pendant  │  upper-360° azimuth, elevation up
                       │     Node      │
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

- **mmWave Radar:** Presence/motion of objects approaching the body envelope. Selected for its ability to function in darkness and through obscuration (fog/smoke).
- **IMU (Inertial Measurement Unit):** Critical for determining the orientation of the node relative to the body frame.
- **Microphone Array:** Acoustic localization — direction-of-arrival of sudden sounds (glass break, verbal aggression).
- **Short-range LiDAR / ToF:** Close-range geometry where radar resolution is insufficient.
- **Environmental Sensors:** Temperature, gas, air quality.
- **Haptic Actuator:** Silent directional alerts to the wearer.

The **belt node** is the primary compute and battery — easiest to make heavier without compromising comfort. Other nodes are slim and beam metadata to the belt over a low-power body-area-network radio.

---

## Body-Frame Tracking with PentaTrack

The fusion engine uses [PentaTrack](https://github.com/RebornBeat/PentaTrack) but with the coordinate origin attached to the wearer's *torso*, not the world. Every node's IMU continuously reports its orientation; the fusion layer transforms each node's local detections into a unified body-frame.

This produces a tracked field of objects in body coordinates:

- *"Person approaching from behind, 4 m, closing at 1.5 m/s."*
- *"Vehicle 8 m to the left, parallel motion, 30 km/h."*
- *"Sudden acoustic event 2 m forward-up, glass-break signature, 0.91 confidence."*
- *"Wearer gait deviation: 60% probability of stumble in the next 800 ms."*

PentaTrack's object-type awareness, drift profiles, and anomaly detection are repurposed: the "object types" become "human walking," "human running," "vehicle," "animal," "thrown object" — each with drift profiles tuned for that class's plausible motion in body-frame coordinates.

---

## Research Domain: Extreme Velocity Sensing

A specific track of this project researches the feasibility of detecting high-velocity projectiles (ballistics) using wearable sensors. This is purely theoretical research to define the "Reaction Time Budget."

**The Physics Problem:**
Standard sensing technologies fail at ballistic speeds.
*   **LiDAR:** Too slow (scanning latency creates blind spots between scans).
*   **Acoustic:** Too slow (sound travels slower than bullets; the bullet hits before the sound arrives).

**The Proposed Solution (Research Subject):**
*   **Doppler Radar:** Continuous Wave (CW) radar has zero scanning latency and detects velocity shifts instantaneously.
*   **Event-Based Vision:** Neuromorphic cameras react in microseconds, capturing the trajectory without the bandwidth overhead of standard video.

This research documents the latency budget required to detect a projectile at 800 m/s within a 3-meter engagement zone. It serves as the foundational analysis for any future "upstream" signal that could theoretically trigger a passive defense (e.g., rapid fiber stiffening).

---

## Output Modalities (Strictly Informational)

What SENTINEL-WEAR *does* with what it perceives:

- **Haptic alerts.** Directional buzz on the appropriate node — "approach from your right rear" buzzes the right-rear-most node.
- **Audio alerts** (bone conduction or earpiece, optional).
- **Companion-app overlay** on phone or smartwatch showing the body-frame field as an abstract radar-style display.
- **Logging** for personal review.
- **Optional emergency contact** — manually triggered, never automatic — that sends a position to a designated contact.

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
│   │   ├── body_coordinate_fusion.md
│   │   ├── drift_profiles_at_body_scale.md
│   │   ├── form_factor_human_factors.md
│   │   ├── extreme_velocity_sensing.md   <-- Ballistic detection feasibility
│   │   ├── passive_materials_research.md <-- Spider-silk / UHMWPE context
│   │   └── why_sensing_only.md
│   ├── api/
│   └── assets/
├── hardware/
│   ├── schematic/
│   │   ├── pendant_node/
│   │   ├── bracelet_node/
│   │   ├── belt_node/           # primary compute
│   │   ├── anklet_node/
│   │   └── eyewear_node/
│   ├── gerbers/
│   ├── bom/
│   ├── assembly/
│   ├── datasheets/
│   ├── testing/
│   └── hardware_config.md
├── firmware/
│   ├── src/
│   │   ├── drivers/
│   │   ├── logic/
│   │   ├── ban_protocol/        # body-area-network radio
│   │   └── main.c
│   ├── bootloader/
│   ├── tools/
│   ├── platformio.ini
│   └── firmware.md
├── software/
│   ├── belt_controller/         # primary fusion + PentaTrack runner
│   ├── mobile/                  # iOS / Android companion
│   ├── desktop/                 # config / log review
│   ├── cli/
│   ├── sdk/
│   ├── protocol/
│   │   └── protocol_spec.md     # body-area mesh protocol
│   └── software.md
├── mechanical/
│   ├── cad/                     # jewelry-grade enclosures
│   ├── stl/
│   ├── drawings/
│   └── mechanical_spec.md       # hypoallergenic materials, water resistance
├── production/
├── legal/
│   ├── compliance.md            # FCC for radios, jewelry-metal regulations
│   ├── medical_device_disclaimer.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

The `mechanical/` directory is unusually important for this project — wearable form factor is a research dimension in its own right.

---

## Roadmap

- **Phase 1:** Belt-node-only bench prototype — full sensor stack, body-frame fusion of one node.
- **Phase 2:** Add bracelet + pendant — demonstrate multi-node body-frame fusion and drift correction.
- **Phase 3:** Full mesh of all six nodes.
- **Phase 4:** Comfort, durability, weight, and human-factors study.
- **Phase 5:** Public open-data release of body-frame trajectory dataset for the research community.
- **Parallel Research Track:** Simulation and latency analysis of extreme-velocity (ballistic) detection feasibility.

There is no Phase N for actuation. By design.

---

## Disclaimer (Verbatim)

> SENTINEL-WEAR is a research and education project. It is not a medical device, not a personal protective equipment certification, not a self-defense product, not a substitute for situational awareness, and not a substitute for any law-enforcement, medical, or emergency service. The maintainers make no warranty of fitness for any safety-critical use. Use at your own risk for educational purposes only.

---

## License

MIT for code. CERN-OHL-S v2 for hardware. CC BY 4.0 for documentation.
