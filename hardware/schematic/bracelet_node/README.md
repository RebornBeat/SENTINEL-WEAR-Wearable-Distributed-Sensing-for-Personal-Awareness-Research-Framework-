# Bracelet Node — SENTINEL-WEAR

**Form Factor:** Jewelry-grade wearable (wrist/forearm mount)
**Role:** Forearm-frame distributed sensing + Haptic alert output
**Status:** Research / Reference Design

---

## 1. Purpose

The Bracelet Node is one of the primary sensing and interaction nodes in the SENTINEL-WEAR mesh. Worn on the wrist or forearm, it provides:

*   **Sensing:** Covers the forward and lateral hemispheres relative to the wearer’s arm.
*   **Body-Frame Contribution:** The node’s IMU data feeds the multi-IMU body-pose estimator, contributing to the overall body-frame stabilization.
*   **Alerting:** The primary haptic actuator for directional alerts is housed here. When an object approaches from the wearer’s left side, the left bracelet fires the haptic pattern.

---

## 2. Architectural Capabilities

The architecture imposes no limits on the specific components used. The following describes the *functional categories* a Bracelet Node typically integrates. The specific sensor models, power budget, and form factor are determined by the research configuration and available hardware.

### Sensing

*   **mmWave Radar:** Primary modality for detecting presence, velocity (via Doppler), and micro-Doppler activity signatures (human vs. animal vs. vehicle). The radar illuminates the forearm-frame hemisphere.
*   **IMU (6-axis or 9-axis):** Provides the node’s orientation relative to the body frame. The orientation is critical for transforming detections from the node frame to the torso frame.
*   **Short-range LiDAR / ToF (Optional):** For close-range geometry refinement where radar resolution is insufficient.

### Output

*   **Haptic Actuator:** A linear resonant actuator (LRA) or eccentric rotating mass (ERM) motor for directional haptic alerts. The haptic encoder in the firmware generates patterns that encode alert class and urgency.

### Communication

*   **Body-Area Network (BAN) Radio:** BLE 5.x or UWB for low-latency communication with the belt controller. Transmits processed detection metadata (not raw sensor data) and receives haptic commands.

### Power

*   **Battery:** Rechargeable Li-Po, sized according to the user’s runtime requirements and sensor complement.
*   **Charging:** Magnetic pogo pins or wireless charging coil, compatible with the chosen enclosure design.

---

## 3. Connectivity-Agnostic Design

The Bracelet Node is a component of a distributed sensing mesh. Its communication is not restricted to a specific back-end:

*   **Local-Only:** Can operate entirely offline, transmitting only to the belt controller for body-frame fusion.
*   **Cloud-Connected:** Can be configured (via the belt controller) to relay metadata to a cloud endpoint.
*   **Hybrid:** Any combination.

**No data limits are imposed.** If the research configuration requires logging raw radar IQ data or IMU samples at 1kHz to a local buffer or external storage, the hardware design should accommodate the necessary flash/RAM or expose the interface to an external logger.

---

## 4. Sensing vs. Identification

The Bracelet Node is part of the **Sensing Layer**.

*   **It provides physics-based identification:**
    *   Radar micro-Doppler → Activity class (walking, running).
    *   Radar/LiDAR geometry → Size class.
*   **It does NOT include an identification sensor (camera/biometric).**
    *   The Identification Layer is reserved for the Pendant Node (opt-in).
    *   The Bracelet Node is camera-free by architecture, ensuring no privacy-invasive modality is present in this part of the mesh.

**Therefore, no hardware kill-switch for an identification sensor is required on this node.** A power switch is recommended for user control, but not a camera-specific interlock.

---

## 5. Design Philosophy: Limitless Configuration

This reference design does not constrain:

*   **Sensor Specifications:** Resolution, range, and update rate are determined by the selected components.
*   **Power Budget:** User-specified based on desired runtime and sensor load.
*   **Form Factor:** Can be miniaturized to jewelry-scale or expanded for research bench testing.

If the research path identifies a need for higher-performance sensors, the PCB footprint and power subsystem should be adapted accordingly. The firmware architecture (Rust, `no_std`) is modular and can drive a wide range of sensors via the OMNI-SENSE trait interfaces.

---

## 6. Directory Contents

This directory contains the hardware design files for the Bracelet Node reference implementation.

```
bracelet_node/
├── bracelet_node.kicad_pro      # KiCad project file
├── bracelet_node.kicad_sch      # Main schematic
├── bracelet_node.kicad_pcb      # PCB layout
├── bracelet_node.kicad_prl      # KiCad project settings
├── fp-info-cache                # Footprint cache
├── fp-lib-table                 # Footprint library table
├── sym-lib-table                # Symbol library table
├── README.md                    # This file
└── gerbers/                     # Generated manufacturing files (if present)
    ├── *.gbr                    # Gerber files
    └── *.drl                    # Drill files
```

*The Gerber files may not be present in the initial commit. They are generated from the KiCad PCB file.*

---

## 7. Related Documentation

*   **Firmware:** `firmware/src/bin/bracelet_left.rs` / `bracelet_right.rs`
*   **Sensing Pipeline:** `crates/sentinel-perception/src/radar_pipeline.rs`, `imu_pipeline.rs`
*   **Haptic Logic:** `crates/sentinel-alerts/src/haptic_encoder.rs`
*   **System Architecture:** `docs/theory/body_coordinate_fusion.md`
