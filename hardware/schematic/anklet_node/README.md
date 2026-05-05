# Anklet Node — Hardware Schematic Reference Design

**Node Position:** Lower leg / above ankle — worn on both left and right ankles
**Sensor Coverage:** Ground plane, low-elevation detection, gait analysis
**Status:** Reference design, variant 1 of N.

---

## Purpose

The anklet nodes are the ground-level awareness layer. Their position close to the ground provides:
- Detection of floor-level objects (pets, low-profile intruders, trip hazards)
- Ground-plane coverage for approaching targets below the knee
- IMU for leg-segment pose and the primary input for gait analysis
- Haptic alerts for ground-level threats or directional cues from below

The distinctive motion profile of anklet nodes during walking (large, periodic oscillations) makes them excellent calibration anchors for the walk-through calibration process.

---

## Candidate Component Set (Test Variants — Not Locked)

### MCU
- **Variant A:** Nordic nRF5340 (consistent with bracelet and pendant)
- **Variant B:** Silicon Labs EFR32BG24 (lower power for longer battery in ankle form factor)

### Short-Range Sensing
Anklets are constrained by proximity to the ground and body. Long-range radar is less useful here.
- **Variant A:** Acconeer XR112 (ultra-compact, 60 GHz, short-to-mid range)
- **Variant B:** 1D ToF sensor (STMicro VL53L5CX, 8×8 matrix, 6 m range, for foot-forward direction sensing)
- **Variant C:** Short-range PIR (detection of warm bodies at ground level — choke-point style)

Note: anklet radar orientation is predominantly forward (in the direction of walking) and slightly upward (to detect approaching bodies before they reach the wearer's feet).

### IMU
- **Variant A:** Bosch BMI270 (consistent across all nodes)
- **Variant B:** ST LSM6DSV (iNEMO 7-axis, built-in machine learning for step detection)

IMU at ankle is the highest-dynamic-range node — it must handle impact (heel strike), rotation, and large angular excursions during normal walking.

### Haptic Actuator
- **Variant A:** Precision Microdrives C10-100 LRA (as bracelet)
- Haptic patterns at ankle are used for: directional alerts from below (ground-level approach), gait warnings (stumble pre-alert)

### BAN Radio
- BLE 5.3 via MCU

### Battery
- LiPo 3.7V, 300–500 mAh (larger battery acceptable at ankle; less jewelry-constrained)
- Target 48–72 hour runtime

---

## Gait Analysis Integration

The anklet IMU data is the primary input for `sentinel-body-frame::gait_analyzer`. Key gait features extracted from ankle IMU:
- Step frequency (cadence)
- Step regularity (gait coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right foot comparison)
- Pre-stumble signature: asymmetric loading + balance recovery micro-accelerations

These features feed PentaTrack's `WearerSelfMotion` anomaly class.
