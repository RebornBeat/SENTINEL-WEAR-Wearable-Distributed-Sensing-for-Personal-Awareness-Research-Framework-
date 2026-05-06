# Test Jig PCB — SENTINEL-WEAR Production Testing

**Project:** SENTINEL-WEAR
**Purpose:** Automated validation for all SENTINEL-WEAR node types and all variants.
**Status:** Reference Design

---

## 1. Overview

The SENTINEL-WEAR test jig validates wearable nodes before deployment. It covers electrical validation, functional sensor testing, data handling mode verification, optional hardware switch testing (if installed), IP67 verification, haptic pattern testing, and firmware verification.

**All node variants are tested through the same carrier board by swapping node-specific test shields.**

---

## 2. Hardware Architecture

### Carrier Board (Universal)

| Component | Specification | Purpose |
|---|---|---|
| Test Controller MCU | STM32H7 or RP2040 | Real-time signal generation and capture |
| Host SBC | Raspberry Pi 5 or x86 | Runs test suite, logs results, serves UI |
| Programmable Power | 0–5V, 0–3A source/measure | Node power profiling |
| Current Sense | INA219 × 2 (main rail + V_SENSOR) | Power measurement per rail |
| Comms | USB, UART, SWD, I2C, SPI | Firmware flashing, debug, API |
| RF Shield | Aluminum can | Isolates DUT during radar tests |

### Node-Specific Test Shields

| Shield | Key Equipment |
|---|---|
| Pendant Standard Shield | LRA reference microphone, radar reflector, camera chart |
| Pendant 360° Shield | LED array at 8 angular positions, camera calibration chart (curved), stitching verification target |
| Bracelet Shield | LRA resonance fixture, radar absorber, camera test target (Var C) |
| Belt Shield | Wi-Fi test AP, BAN golden node, thermal load fixture |
| Anklet Shield | ToF calibrated target (0.3 m), LRA resonance fixture, IMU impact simulator |
| Eyewear Shield | Event camera LED strobe (< 100 μs), IMU gimbal, forward visual target |

---

## 3. Test Sequences

### 3.1 All Nodes — Mandatory

1. Power-on self-test — MCU boots, firmware CRC verified.
2. Rail verification — all power rails within ±5% of nominal.
3. Sensor enumeration — all I2C/SPI sensors respond to init.
4. IMU gravity vector — reads 9.81 ± 0.2 m/s² at rest.
5. BAN radio ping — appears on network, responds within 200 ms.
6. Battery level read — 4.1–4.2 V on full charge.
7. Firmware version — matches current published build hash.

### 3.2 Pendant Node — Standard and Medallion

1. Radar detection at 0.5 m test target — detection event generated within 500 ms.
2. Acoustic DOA — reference speaker at 90°. Error < ±15°.
3. **Camera (if installed): Data handling mode verification:**
   - Metadata-only: only classification tag transmitted; no file created on SD.
   - Local storage: recording file created on SD within 500 ms of trigger.
   - App streaming: belt controller receives video stream within 2 seconds.
4. **Optional hardware switch test (only if switch is physically installed on this unit):**
   - Switch DISABLED: confirm V_SENSOR rail = 0 V ± 0.05 V. If V_SENSOR > 0.1 V: FAIL.
   - Switch ENABLED: confirm V_SENSOR > 3.0 V.

### 3.3 Pendant Node — 360° Curved

1. All standard pendant tests above.
2. **Camera array initialization:** All N cameras initialize and produce frames within 5 seconds.
3. **Hardware sync:** FSYNC line verified — all cameras trigger within ±1 ms of each other.
4. **Stitching quality:** LED array at 8 positions. Straight reference lines continuous across camera boundaries. Seam error < 3 pixels at output resolution.
5. **360° recording test:** 30-second equirectangular recording stored to SD card. File size > 0 bytes. SHA-256 computed and stored in integrity manifest.
6. **Companion app stream:** Belt node receives stitched 360° stream. Companion app displays 360° view within 5 seconds of activation.

### 3.4 Bracelet Nodes

1. Radar detection at 0.3 m test target.
2. **Haptic pattern test:**
   - Play reference waveform.
   - Measure with calibrated microphone.
   - Verify LRA resonance at 150 ± 15 Hz (or ERM variant spec).
   - Motor current within spec (300–500 mA peak full-amplitude LRA).
3. **Camera (Variant C, if installed): data handling mode verification** (as pendant above).
4. **Optional hardware switch (if installed):** Same test as pendant.

### 3.5 Belt Node

1. Wi-Fi AP detection — belt node appears on test network.
2. BAN-to-Wi-Fi relay — packet injected via BAN, relayed to Wi-Fi within 200 ms.
3. API server — companion app test client connects, all basic endpoints respond 200 OK.
4. Fusion computation benchmark — 1000-frame PentaTrack update. Target: < 10 ms/frame.
5. Recording manager test — inject `RecordingAvailable` event; verify recording indexed and accessible via API.
6. Current draw at full compute load — within thermal spec.

### 3.6 Anklet Nodes

1. ToF reading at calibrated 0.3 m target — within ±5 cm.
2. IMU heel-strike simulation — verify ≥ 3 g recorded.
3. Haptic resonance test — as bracelets.
4. Gait algorithm smoke test — inject 10 simulated steps; verify step count = 10.
5. **Camera (Variant B, if installed):** data handling mode verification.

### 3.7 Eyewear Node

1. Event sensor response to LED strobe (< 100 μs) — event burst received within 1 ms.
2. IMU at 200 Hz ODR — no dropped samples over 10 seconds.
3. **Camera (Variants B/C, if installed):** image quality check; data handling mode verification.

---

## 4. IP67 Test Fixture

Submersion fixture: 1 m water depth, 30 minutes.

**Post-submersion:**
- Node powers on within 60 seconds.
- All sensors initialize.
- BAN radio appears on network.
- No visible moisture inside transparent test window.
- Rail voltages within ±2% of pre-submersion values.

---

## 5. Software

**Test runner:** Rust CLI (`sentinel-test-harness` crate). Results as JSON with per-test pass/fail, measured values, timestamps.

**Example result (Pendant 360°):**
```json
{
  "serial": "SW-PND360-00012",
  "node_type": "Pendant360",
  "firmware_rev": "v1.0.0",
  "tests": [
    {"name": "Power_Rails", "status": "PASS", "value": "3.31V"},
    {"name": "Camera_Array_Init", "status": "PASS", "cameras_found": 8},
    {"name": "Sync_Accuracy", "status": "PASS", "max_delta_ms": 0.4},
    {"name": "Stitch_Quality", "status": "PASS", "max_seam_error_px": 2.1},
    {"name": "Recording_360", "status": "PASS", "file_size_bytes": 45231088},
    {"name": "BLE_Link", "status": "PASS", "rssi_dbm": -41}
  ],
  "final_result": "PASS"
}
```

---

## 6. Safety

- Battery safety: proper charge profiles enforced by test jig. Do not leave unattended.
- ESD precautions mandatory when handling bare PCBs.
- Optional hardware switch test: only performed on units where the switch is physically installed. Units without hardware switches proceed directly to software data handling mode tests.
