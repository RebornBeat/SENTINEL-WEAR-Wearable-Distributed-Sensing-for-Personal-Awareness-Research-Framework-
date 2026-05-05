# Pendant Node — Hardware Schematic Reference Design

**Node Position:** Neck/chest — worn as a necklace or pendant
**Sensor Coverage:** Upper hemisphere (360° azimuth, upward elevation)
**Status:** Reference design, variant 1 of N. All component choices are test candidates.

---

## Purpose

The Pendant Node is the highest-information-value wearable node. Its chest/neck position gives it 360° azimuthal view of the environment at chest height — the most relevant sensing plane for approaching humans and objects. It also hosts:
- The primary acoustic sensor for sound-event direction and material classification
- The optional identification sensor (visual/biometric — opt-in, kill-switch gated)
- IMU for body-frame pose contribution

---

## Candidate Component Set (Test Variants — Not Locked)

### MCU
- **Variant A:** Nordic nRF5340 (Cortex-M33 app + M33 net, BLE 5.3, USB, excellent power management)
- **Variant B:** STM32WB55 (Cortex-M4 + M0+, BLE 5.0, ultra-low power, 2.4 GHz)

The pendant MCU handles: radar processing, IMU filter, acoustic beamforming (partial), BAN radio, and (when opted-in) identification sensor interface. Compute requirements are moderate.

### mmWave Radar
- **Variant A:** Texas Instruments IWR6843ISK (60 GHz, evaluation module form factor — for early testing)
- **Variant B:** Acconeer XR112 module (60 GHz, SPI, ultra-compact — for wearable integration)
- **Variant C:** Infineon BGT60ATR24C (60 GHz, MMIC, requires external antenna design)

Coverage: antenna oriented to maximize forward and lateral hemisphere coverage from chest position.

### IMU
- **Variant A:** Bosch BMI270 (6-axis, OIS, ultra-low power, SPI/I2C, 0.1 mg noise density)
- **Variant B:** Invensense ICM-42688-P (6-axis, 0.07 mg noise density, SPI, smallest package)
- **Variant C:** ST LSM6DSV (6-axis + temperature, iNEMO 7-axis, FIFO, SPI)

IMU is critical for body-frame drift correction. Noise density and bias stability are the primary selection criteria.

### Microphone Array (Acoustic)
- **Variant A:** Knowles SPH0645LM4H × 3 (I2S PDM, compact, 65 dBA SNR) — 3-element planar array
- **Variant B:** ST MP23DB01HP × 4 (analog, 64 dBA SNR) — 4-element tetrahedral on flexible PCB

Array configuration: planar 3-element for space-constrained pendants; tetrahedral 4-element for better 3D direction-of-arrival in belt-form pendant variants.

### Identification Sensor (Opt-In Only — Kill Switch Required)
All variants under research. None mandated. The kill switch is mandatory before any variant can be activated.

- **Variant A:** OmniVision OV2640 (2 MP, small package, MIPI/DVP) — camera-based face detection
- **Variant B:** Luxonis OAK-D Lite (stereo + neural inference on-module) — self-contained identification
- **Variant C:** HiMax HM01B0 (QVGA, 2.1 mW, ultra-low power) — minimal footprint, first-stage detection
- **Variant D:** None — pendant in audio-only identification mode using voice biometrics

Selection criteria: classification accuracy at 1–3 m, on-device inference capability (no raw video off-module), power consumption impact on battery life.

### BAN Radio
- **Variant A:** BLE 5.3 via nRF5340 network core (if MCU is nRF5340)
- **Variant B:** UWB via Qorvo DW3000 (sub-ns timestamping, <1 m ranging accuracy, higher power)

BLE: adequate latency for awareness alerts. UWB: required if precise ranging or sub-millisecond sync is needed (research variant).

### Battery
- **Target:** LiPo 3.7V, 150–300 mAh (jewelry weight budget: 10–15 g battery maximum)
- **Variants under test:** 150 mAh (ultra-thin, 1.5 mm), 200 mAh (standard pendant thickness), 300 mAh (bulkier pendant, 5 mm)

---

## Form Factor Constraints

Weight budget: 30–60 g total including enclosure, battery, and all electronics (see `docs/theory/form_factor_human_factors.md`).

PCB dimensions: approximately 30 mm × 25 mm for electronics (multiple PCB stacking acceptable for depth).

Charging: USB-C or magnetic POGO pin (compatible with jewelry wear and water resistance).

Water resistance: IP54 minimum (sweat, rain); IP67 target (hand washing, brief immersion).

---

## Kill Switch Implementation

The identification sensor kill switch is a physical slider integrated into the pendant enclosure. When in DISABLED position:
- Power rail to identification sensor is cut by hardware (MOSFET gate controlled by slider position)
- Data lines (MIPI/USB) are disconnected by hardware (analog switches in series)
- MCU GPIO reads kill-switch state at boot; if DISABLED, `ImagingSensor::initialize()` returns `KillSwitchEngaged`

The slider position must be visible to the wearer without removing the pendant.
