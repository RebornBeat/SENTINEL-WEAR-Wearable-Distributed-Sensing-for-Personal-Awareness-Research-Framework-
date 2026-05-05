# Bracelet Node — Hardware Schematic Reference Design

**Node Position:** Wrist/forearm — worn on both left and right wrists
**Sensor Coverage:** Forearm hemisphere (forward of body, arm-direction dependent)
**Status:** Reference design, variant 1 of N.

---

## Purpose

The bracelet nodes are the primary directional alert delivery nodes. The wrists face outward during normal posture and arm swing, making them ideal for:
- Haptic alert delivery (the node closest to the approach direction buzzes)
- Forearm-hemisphere radar coverage
- IMU for arm-segment body-frame contribution and gesture detection

Two bracelet nodes (left and right wrist) together cover the lateral body hemispheres and improve forward/rear directional discrimination.

---

## Candidate Component Set (Test Variants — Not Locked)

### MCU
- **Variant A:** Nordic nRF5340 (as pendant — consistent development across nodes)
- **Variant B:** Silicon Labs EFR32BG24 (BLE 5.3, ultra-low power, Cortex-M33)

Bracelet MCU compute requirements are lower than pendant. The primary tasks are: radar polling, IMU Madgwick filter, haptic control, BAN radio.

### mmWave Radar
- **Variant A:** Acconeer XR112 (60 GHz, SPI, low power, compact) — preferred for wrist form factor
- **Variant B:** Infineon BGT60ATR24C with custom 2-element patch antenna

Antenna orientation: PCB antenna arrays oriented toward the palm/dorsal face of the wrist. The goal is to maximize coverage of the hemisphere in front of and to the side of the bracelet-wearing arm.

### IMU
- **Variant A:** Bosch BMI270 (as pendant — consistent calibration)
- **Variant B:** Invensense ICM-42688-P (smallest package option)

### Haptic Actuator
- **Variant A:** Precision Microdrives C10-100 linear resonant actuator (LRA, 10 mm diameter, 150 Hz resonance)
- **Variant B:** Precision Microdrives 307-103 (ERM, vibration motor, wider frequency range)
- **Variant C:** AAC Technologies ALPS HF2-C3 (thin LRA, 2 mm z-height, for ultra-slim bracelet)

LRA (Linear Resonant Actuator) preferred over ERM for distinct directional patterns and lower power. Haptic driver IC: Texas Instruments DRV2605L (I2C, waveform library, auto-resonance detection).

### BAN Radio
- BLE 5.3 via MCU integrated radio

### Battery
- LiPo 3.7V, 200–400 mAh
- Target 24–72 hour runtime between charges

### Charging
- Magnetic pogo-pin charging contacts on inner wrist surface (skin-contact side)

---

## Bracelet-Specific Design Notes

**Form factor:** Bracelet must sit comfortably against the wrist without impeding wrist flexion. PCB should be placed in the dorsal (back-of-hand) facing section of the band. Sensor apertures face away from wrist.

**Antenna design:** mmWave antennas at 60 GHz are small (few mm) and can be integrated into the PCB silkscreen area. The antenna patch should face outward (away from wrist) to avoid body absorption.

**Water resistance:** IP67. Sweat, rain, hand washing are guaranteed exposure conditions.

**Material contact:** All skin-contact surfaces must use hypoallergenic materials: titanium, surgical steel, medical-grade silicone, or hard-coated aluminum (EN 1811 nickel release compliance).
