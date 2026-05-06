# Anklet Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Lower leg / above ankle — worn on both left and right ankles
**Primary Role:** Ground-plane sensing, gait analysis, lower-hemisphere coverage, haptic directional alerts
**Node Class:** Sensing-only (no identification sensor)
**Status:** Reference Design

---

## 1. Purpose

The Anklet Node is responsible for sensing the lower hemisphere of the wearer's body-frame. It tracks ground proximity, detects low obstacles, contributes heavily to gait analysis (leg swing phase), and serves as the primary haptic output for threats approaching from the rear or low angles.

Its position on the lower leg provides a unique geometric vantage point: it can see under furniture, sense floor-level movement, and detect ground impact forces directly through the IMU. The distinctive motion profile during walking (large, periodic oscillations) makes anklet nodes excellent calibration anchors for the walk-through calibration process.

---

## 2. Sensor Stack

Optimized for its lower-body role. **This is a reference configuration; the architecture imposes no limits on substitution or upgrades.**

| Component | Role | Typical Part Class | Notes |
|---|---|---|---|
| **Short-Range LiDAR / ToF** | Ground clearance, floor-level geometry | VL53L5CX, TFMini-S, TF-Luna | Direct measurement of distance to floor/obstacles |
| **IMU (6-axis)** | Leg swing, gait phase, orientation | BMI270, ICM-42688-P | Detects step events, stride length, kick gestures |
| **Haptic Actuator** | Directional alerts | LRA or ERM, 8–10 mm diameter | Buzz for rear/side approach alerts |
| **BAN Radio** | Body-area network comms | BLE 5.2+ SoC (nRF5340, EFR32BG24) | Low-power advertising + burst transfer |
| **Environmental (Optional)** | Temp/Humidity | BME280, SHT4x | Context for gait analysis |
| **Battery** | Power | LiPo pouch 150–500 mAh | Sized for 24–72 hour active wear |

**Constraints:**
- **No Camera:** Sensing node only. No identification hardware.
- **Form Factor:** Must fit within a profile that doesn't snag on pants, furniture, or brush.

---

## 3. Candidate Component Set (Test Variants — Not Locked)

### MCU

- **Variant A:** Nordic nRF5340 (consistent with bracelet and pendant for firmware uniformity)
- **Variant B:** Silicon Labs EFR32BG24 (lower power for longer battery life in ankle form factor)

### Short-Range Sensing

- **Variant A:** Acconeer XR112 (ultra-compact, 60 GHz, SPI, short-to-mid range radar)
- **Variant B:** STMicro VL53L5CX (8×8 zone ToF matrix, 6 m range, I2C) — for foot-forward direction sensing
- **Variant C:** Short-range PIR (detection of warm bodies at ground level — choke-point style)

**Antenna/sensor orientation:** Predominantly forward (direction of walking) and slightly upward (to detect approaching bodies before they reach the wearer's feet).

### IMU

- **Variant A:** Bosch BMI270 (consistent across all nodes)
- **Variant B:** ST LSM6DSV (iNEMO 7-axis, built-in machine learning for step detection)

IMU at ankle is the highest-dynamic-range node — must handle heel-strike impact (~5–10 g peak), large rotation, and angular excursions during normal walking.

### Haptic Actuator

- **Variant A:** Precision Microdrives C10-100 LRA (10 mm diameter, 150 Hz resonance) — consistent with bracelet
- **Variant B:** ERM (wider frequency range, simpler driver)

Haptic patterns at ankle: directional alerts from below (ground-level approach), gait warnings (stumble pre-alert), low-battery vibration.

### BAN Radio

- BLE 5.3 via MCU integrated radio (primary)

### Battery

- LiPo 3.7V, 300–500 mAh (larger battery acceptable at ankle — less jewelry-constrained than pendant/bracelet)
- Target 48–72 hour runtime

---

## 4. Mechanical Design

### Placement

- Worn on the lower leg, typically superior to the malleolus (ankle bone).
- Secured by a strap or integrated band.
- Can be medial or lateral; calibration accounts for left/right asymmetry.

### Enclosure

- **Material:** Hypoallergenic polymer or metal alloy (titanium, surgical steel).
- **IP Rating:** Minimum IP67 (water/sweat resistance). IP67 for immersion tolerance.
- **Finish:** Rounded edges to avoid skin irritation during running.
- **Weight Target:** < 80–100 g total (including battery).
- **Skin Contact:** Nickel-free alloys. Ventilation slots to reduce sweat accumulation.

---

## 5. Electrical Design

### Power Architecture

| Rail | Voltage | Source | Usage |
|---|---|---|---|
| `V_BAT` | 3.7–4.2V | LiPo battery | Direct to radio PA, haptic peak |
| `V_MCU` | 3.3V | LDO from V_BAT | MCU and sensors |
| `V_SENSOR` | 3.3V / 1.8V | Regulated | ToF, IMU |

- **Charging:** Qi wireless charging or magnetic POGO pin dock.
- **Fuel Gauge:** I2C fuel gauge IC (e.g., MAX17048) for accurate % reporting.

### Interfaces

| Interface | Function |
|---|---|
| **I²C** | IMU, Environmental sensors, Haptic driver |
| **UART / SPI** | ToF/LiDAR (depending on module) |
| **GPIO** | Haptic PWM, Button (reset/pairing) |
| **RF Antenna** | Integrated ceramic or PCB trace antenna |

### Debug

- 4-pin debug header (SWD + UART TX) for firmware flashing.
- Test points: Battery voltage, IMU interrupt, ToF trigger.

---

## 6. Firmware Interface

The firmware (`firmware/src/bin/anklet_left.rs`, `anklet_right.rs`) expects:

- **IMU:** High-ODR (≥ 200 Hz) for gait resolution.
- **ToF:** Polling at 10–20 Hz during normal walk; burst at 50 Hz during sprint detection.
- **Haptic:** Immediate response to `HapticCommand` from belt controller.

**Key Functions:**
- `poll_ground_clearance()` — Returns distance to floor.
- `get_gait_event()` — Returns step/kick/stance phase event.
- `fire_haptic(pattern, intensity)` — Drives actuator with pattern and amplitude.

---

## 7. Gait Analysis Integration

The anklet IMU data is the primary input for `sentinel-body-frame::gait_analyzer`. Key features extracted:

- Step frequency (cadence)
- Step regularity (gait coefficient of variation)
- Heel-strike impact magnitude
- Stride asymmetry (left vs. right foot comparison)
- Pre-stumble signature: asymmetric loading + balance recovery micro-accelerations

These features feed PentaTrack's `WearerSelfMotion` anomaly class and are used to predict stumble events before they become falls.

---

## 8. Calibration

Anklet nodes require geometric calibration relative to the belt node:
- **Neutral Pose Calibration:** Establishes local "down" vector with wearer standing still.
- **Walk-Through Calibration:** Trilaterates precise position on the leg relative to torso origin.

No calibration data stored on the node; it streams raw IMU data to belt controller during calibration mode.

---

## 9. Testing & Validation

**Functional Tests:**
- Ground detection accuracy: target 0.1–1.5 m range.
- IMU range and noise floor at rest and heel-strike.
- Haptic intensity: must be felt through clothing and a sock.

**Environmental Tests:**
- IP67 water immersion (1 m, 30 minutes).
- Sweat/salt spray exposure.
- Impact testing: simulated kick against furniture (10 g impact event).

**Wear Testing:**
- 24-hour wear comfort with no pressure points.
- Snag resistance on stairs, brush, carpet.
- Charging reliability after 100 cycles.

---

## 10. Reference BOM (Example)

| Component | Part Number | Qty | Notes |
|---|---|---|---|
| BLE SoC | nRF5340 | 1 | ARM Cortex-M33, BLE 5.2 |
| IMU | ICM-42688-P | 1 | 6-axis, low noise |
| ToF Sensor | VL53L5CX | 1 | 8×8 zone, up to 4m |
| LRA Driver | DRV2605L | 1 | I2C haptic driver |
| LRA Actuator | C10-100 | 1 | 10 mm, linear resonant |
| Fuel Gauge | MAX17048 | 1 | LiPo % reporting |
| Battery | 301230 LiPo | 1 | 120–300 mAh pouch |
| Antenna | Johanson 2450AT18A100 | 1 | 2.4 GHz chip antenna |

---

## 11. Design Notes

- **Left vs Right:** Hardware is identical; firmware (`anklet_left` vs `anklet_right`) handles coordinate mapping and haptic direction encoding.
- **Synchronization:** Time sync with belt node is critical for gait phase accuracy. Use BLE Connection Event timestamps for sub-millisecond alignment.
- **Dual-Purpose:** May act as secondary receiver for body-area-network time sync reference.
