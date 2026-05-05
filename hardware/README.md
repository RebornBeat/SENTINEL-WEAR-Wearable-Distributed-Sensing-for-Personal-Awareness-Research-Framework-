# Hardware — SENTINEL-WEAR

This directory contains the hardware reference designs for SENTINEL-WEAR's sensing nodes: pendant, bracelet, belt, anklet, and (optional) eyewear form factors. Each node carries some subset of mmWave radar, IMU, microphone array, short-range LiDAR/ToF, environmental sensors, BAN radio, battery, and haptic actuator as appropriate to its form factor.

The hardware in this directory is **sensing-only**. No design in this directory implements:

- Active emitters above eye-safe educational classes.
- RF emitters operating outside FCC Part 15 / CE RED unregulated bands.
- Mechanical actuators capable of acting against any external object.
- Energy storage at densities or chemistries beyond standard wearable lithium-polymer.
- Any mechanism intended to provide protective function in the personal-protective-equipment sense.

Researchers contributing hardware designs are asked to confirm in their pull requests that the contribution stays within these bounds. PRs proposing active-response hardware will be closed and the contributor referred to the future-research document for the appropriate research-domain context.

See `legal/compliance.md`, `legal/research_ethics.md`, and `legal/export_control_posture.md` for the project's full posture.
