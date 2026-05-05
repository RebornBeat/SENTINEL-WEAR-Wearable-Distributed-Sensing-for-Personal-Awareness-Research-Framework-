# Hardware — SENTINEL-WEAR Anklet Node

**Form Factor:** Jewelry-grade anklet / lower-leg band
**Primary Role:** Ground-plane sensing, gait analysis, lower-hemisphere coverage, rear/side approach alerts
**Node Class:** Sensing-only (no identification sensor)

---

## 1. Purpose

The Anklet Node is responsible for sensing the lower hemisphere of the wearer’s body-frame. It tracks ground proximity, detects low obstacles, contributes heavily to gait analysis (leg swing phase), and serves as a primary haptic output for threats approaching from the rear or low angles.

Its position on the lower leg gives it a unique geometric vantage point: it can see under furniture, sense floor-level movement, and detect ground impact forces directly through the IMU.

---

## 2. Sensor Stack

The Anklet Node carries a focused sensor set optimized for its lower-body role. This is a **reference configuration**; the architecture imposes no limits on substitution or upgrades.

| Component | Role | Typical Part Class | Notes |
| :--- | :--- | :--- | :--- |
| **Short-Range LiDAR / ToF** | Ground clearance, floor-level geometry | Single-point or 1D ToF module (e.g., VL53L5CX, TFMini-S) | Direct measurement of distance to floor/obstacles. |
| **IMU (6-axis)** | Leg swing, gait phase, orientation | BMI270, ICM-42688-P | Detects step events, stride length, kick gestures. |
| **Haptic Actuator** | Directional alerts | LRA or ERM, 8–10mm diameter | Provides buzz for rear/side approach alerts. |
| **BAN Radio** | Body-area network comms | BLE 5.2 SoC (nRF5340, ESP32-C6) | Must support low-power advertising + burst transfer. |
| **Environmental (Optional)** | Temp/Humidity | BME280, SHT4x | Context for gait analysis (heat/humidity). |
| **Battery** | Power | LiPo pouch (100–200 mAh typical) | Sized for 12–24 hour active wear. |

**Constraints:**
- **No Camera:** This is a sensing node only.
- **Form Factor:** Must fit within a profile that doesn’t snag on pants, furniture, or brush.

---

## 3. Mechanical Design

### Placement
- Worn on the lower leg, typically **superior to the malleolus (ankle bone)**.
- Secured by a strap or integrated band.
- Can be medial or lateral; calibration expects left/right asymmetry.

### Enclosure
- **Material:** Hypoallergenic polymer or metal alloy (titanium, surgical steel).
- **IP Rating:** Minimum IP67 (water/sweat resistance).
- **Finish:** Rounded edges to avoid skin irritation during running.
- **Weight Target:** < 80g total (including battery).

### Skin Contact
- Requires nickel-free alloys if metal backplate contacts skin.
- Ventilation slots to reduce sweat accumulation.

---

## 4. Electrical Design

### Power Architecture
- **Primary Power:** Single-cell LiPo.
- **Charging:** Qi wireless charging or magnetic pogo-pin dock.
- **Regulation:** Buck/boost for 3.3V and 1.8V rails.
- **Battery Monitoring:** Fuel gauge IC for accurate % reporting to belt controller.

### Interfaces
| Interface | Function |
| :--- | :--- |
| **I²C** | IMU, Environmental sensors |
| **UART / SPI** | ToF/LiDAR (depending on module) |
| **GPIO** | Haptic driver (PWM), Button (reset/pairing) |
| **RF Antenna** | Integrated ceramic or PCB trace antenna |

### Debug
- **Debug Header:** 4-pin (SWD, UART) for firmware flashing.
- **Test Points:** Battery voltage, IMU interrupt.

---

## 5. Firmware Interface

The firmware (`firmware/src/bin/anklet_left.rs`, `anklet_right.rs`) expects:

- **IMU:** High-ODR (> 200 Hz) for gait resolution.
- **ToF:** Polling at 10–20 Hz during normal walk; burst at 50 Hz during sprint detection.
- **Haptic:** Immediate response to `HapticCommand` from belt controller.

**Key Functions:**
- `poll_ground_clearance()` — Returns distance to floor.
- `get_gait_event()` — Returns step/kick/stance phase.
- `fire_haptic(pattern)` — Drives actuator.

---

## 6. Calibration

Anklet nodes require geometric calibration relative to the belt node:
- **Neutral Pose Calibration:** Establishes local “down” vector.
- **Walk-Through Calibration:** Trilaterates precise position on the leg relative to torso.

No calibration data is stored on the node; it streams raw sensor data to the belt controller for fusion.

---

## 7. Testing & Validation

**Functional Tests:**
- Ground detection accuracy (0.1–1.5m).
- IMU range and noise floor.
- Haptic intensity vs. comfort (must be felt through clothing).

**Environmental Tests:**
- IP67 water immersion.
- Sweat/salt spray.
- Impact testing (simulated kick against furniture).

**Wear Testing:**
- 24-hour wear comfort.
- Snag resistance.
- Charging reliability.

---

## 8. Reference BOM (Example)

| Component | Part Number (Example) | Qty | Notes |
| :--- | :--- | :--- | :--- |
| BLE SoC | nRF5340 | 1 | ARM Cortex-M33, BLE 5.2 |
| IMU | ICM-42688-P | 1 | 6-axis, low noise |
| ToF Sensor | VL53L5CX | 1 | 8x8 zone, up to 4m |
| LRA Driver | DRV2605L | 1 | Haptic driver |
| LRA Actuator | 0832 LRA | 1 | 8mm, linear resonant |
| Fuel Gauge | MAX17048 | 1 | LiPo gauge |
| Battery | 301230 | 1 | 120mAh LiPo pouch |
| Antenna | Johanson 2450AT18A100 | 1 | 2.4GHz chip antenna |

---

## 9. Design Notes

- **Left vs Right:** Hardware is identical; firmware differentiation (`anklet_left` vs `anklet_right`) handles coordinate mapping.
- **Dual-Purpose:** May act as a secondary receiver for body-area-network time sync.
- **Synchronization:** Time sync with belt node is critical for gait phase accuracy; use BLE Connection Event timestamps.
