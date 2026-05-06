# Test Jig PCB — SENTINEL-WEAR Production Testing

**Project:** SENTINEL-WEAR
**Purpose:** Automated validation for all SENTINEL-WEAR wearable node types.
**Status:** Reference Design

---

## 1. Overview

The SENTINEL-WEAR test jig validates wearable nodes before deployment. Unlike AEGIS-MESH's fixed-installation nodes, wearable nodes must pass additional form-factor tests unique to worn devices:

- Physical dimensional check (weight, thickness, connector position)
- Water resistance validation (IP67 submersion fixture)
- Skin-contact material compliance verification (documentation check against EN 1811)
- Haptic pattern verification (audio microphone + FFT to verify LRA resonance at target frequency)
- Kill switch integrity test (mandatory for all pendant nodes with identification sensor populated)

---

## 2. Hardware Specifications (Test Jig PCB)

The Test Jig PCB is a specialized board that sits between the **Test Controller** and the **Node Under Test (NUT)**.

### 2.1 Test Controller

| Component | Specification | Purpose |
|---|---|---|
| Test Controller MCU | STM32H7 or RP2040 | Real-time signal generation & capture |
| Host Interface | USB-C to PC | Runs test suite, logs data |
| Programmable Power | 0–5 V, 0–3 A | Power profiling (sleep/peak current) |
| Current Sense | INA219 or equivalent | Power measurement |
| GPIO Drivers | High-side switches | Pogo pin control |

### 2.2 Pogo Pin Array

- Layout matches test pads on underside of each node PCB variant.
- **Power Pins:** High-current capacity (≥ 2 A peak for haptic burst).
- **Signal Pins:** Standard spring-loaded pogo pins.
- **Fixture Sense:** Detects mechanical insertion.
- **Kill Switch Actuator:** Solenoid or servo to physically toggle the switch during Identity testing.

### 2.3 Node-Specific Test Shields

Each shield provides the physical environment for testing:

| Shield Type | Key Equipment |
|---|---|
| Pendant Shield | Kill switch actuator, LRA reference test microphone, camera target chart |
| Bracelet Shield | LRA resonance fixture, radar absorber |
| Belt Shield | Wi-Fi test AP, BAN golden node, load resistor |
| Anklet Shield | ToF calibrated target (0.3 m), LRA resonance fixture |
| Eyewear Shield | Event strobe, IMU gimbal |

---

## 3. Test Sequence per Node Type

### All Nodes — Mandatory Sequence

1. **Power-on self-test** — MCU boots, firmware CRC verified.
2. **Rail verification** — Measure 3.3 V / 1.8 V / V_BAT within ±5% of nominal.
3. **Sensor enumeration** — I2C/SPI scan confirms all sensors present and responding.
4. **IMU initialization check** — Calibration output within spec; gravity vector ≈ 9.81 m/s².
5. **BAN radio ping** — Node appears on network, responds within 200 ms.
6. **Battery level read** — Full charge = 4.1–4.2 V.
7. **Firmware version check** — Matches current published build hash.

### Pendant Node (with Identification Sensor)

1. Kill switch **DISABLED** position:
   - Confirm zero current to identification sensor (0 mA ± 2 mA on `V_CAM` rail).
   - If current > 2 mA: **FAIL — REJECT UNIT.**
2. Kill switch **ENABLED** position:
   - Confirm identification sensor current > 10 mA (sensor powered).
3. Firmware kill-switch state report: verify `KillSwitchEngaged` status emitted when switch DISABLED.
4. Acoustic array: DOA test with reference speaker at 90° — verify within ±10°.
5. Radar: Presence detection at 0.5 m test target — verify detection event.

### Bracelet and Anklet Nodes

1. Radar presence detection at 0.3 m test target — verify detection event.
2. **Haptic pattern test:**
   - Play reference waveform.
   - Measure with calibrated microphone.
   - Verify LRA resonance peak at 150 ± 15 Hz (or ERM variant spec).
   - Motor current within spec (300–500 mA peak for full-amplitude LRA).

### Anklet Nodes (Additional)

1. ToF sensor reading at calibrated 0.3 m target — verify within ±5 cm.
2. IMU heel-strike impact simulation — verify ≥ 3 g acceleration recorded.

### Belt Node

1. Wi-Fi AP detection — belt node appears on test network.
2. BAN-to-Wi-Fi relay test — packet injected via BAN, relay verified at Wi-Fi interface.
3. Fusion computation benchmark — run 1000-frame PentaTrack update, measure time per frame. Target: < 10 ms/frame.
4. Current draw at full compute load — verify within thermal spec.

### Eyewear Node

1. Event sensor response to LED strobe flash — verify event burst received within 1 ms.
2. IMU response at 200 Hz ODR — verify no dropped samples.

---

## 4. IP67 Test Fixture

A submersion fixture (1 m water depth, 30 minutes) is used to verify IP67 compliance. Fixtures are designed per node shape profile.

**Post-submersion checks:**
- Node must power on within 60 seconds of removal.
- All sensors must initialize.
- BAN radio must appear on network.
- No visible moisture inside transparent test window.
- No change in rail voltages > ±2%.

---

## 5. Software Stack

### Test Runner

Written in Rust (`aegis-test-harness` pattern, adapted for SENTINEL-WEAR nodes).
Results output as structured JSON with per-test pass/fail, measured values, and timestamps.

**Example output:**
```json
{
  "serial": "SW-PND-00042",
  "node_type": "Pendant",
  "firmware_rev": "v1.0.0",
  "tests": [
    {"name": "Power_Rails", "status": "PASS", "value": "3.31V"},
    {"name": "Kill_Switch_Isolated", "status": "PASS", "value": "0.1mA"},
    {"name": "Kill_Switch_Enabled", "status": "PASS", "value": "18.4mA"},
    {"name": "BLE_Link", "status": "PASS", "rssi_dbm": -42},
    {"name": "Haptic_LRA_Resonance", "status": "PASS", "freq_hz": 151}
  ],
  "final_result": "PASS"
}
```

---

## 6. Safety & Compliance

- **Research Context:** This is a research and QC tool, not a medical device.
- **Battery Safety:** The jig handles lithium batteries. Ensure proper charge profiles and current limits. Do not leave unattended during charging cycles.
- **ESD:** Use standard ESD precautions (wrist strap + conductive mat) when handling bare PCBs.
- **Kill Switch Verification:** This test is **mandatory** for all pendant nodes with the identification module populated. A failed kill-switch test means the node **must not** be deployed. Rework or discard.
