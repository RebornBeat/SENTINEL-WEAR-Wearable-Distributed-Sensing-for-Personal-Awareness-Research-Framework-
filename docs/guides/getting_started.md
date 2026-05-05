# Getting Started — SENTINEL-WEAR

---

## Prerequisites

- Rust 1.75+ stable toolchain
- Embedded target for firmware: `rustup target add thumbv7em-none-eabihf`
- Either: physical wearable nodes OR: PC-only development mode (belt controller simulation)

---

## Step 1: Build

```bash
git clone https://github.com/<user>/sentinel-wear.git
cd sentinel-wear
cargo build --release
```

Produces: `target/release/sentinel-belt-controller` — the main process running on the belt node's compute unit.

For firmware:
```bash
cd firmware
cargo build --release --target thumbv7em-none-eabihf --bin pendant_node
cargo build --release --target thumbv7em-none-eabihf --bin bracelet_left
# etc. for each node
```

---

## Step 2: Run in Simulation Mode (PC)

No hardware required. The belt controller simulates all nodes using OMNI-SENSE simulation implementations.

```bash
cargo run --bin sentinel-belt-controller -- \
  --config examples/single_node_bench.toml \
  --mode simulation
```

The simulation generates a synthetic scenario: a simulated wearer walking at 1.5 m/s with simulated objects (approaching pedestrian, passing vehicle) in the body frame. Watch the console output for `AlertEvent` values showing directional detections.

---

## Step 3: Body-Frame Calibration

### Neutral-Pose Calibration (required first)

Put on all wearable nodes. Stand in a neutral pose: arms relaxed at sides, facing forward, upright.

```bash
curl -X POST http://belt-node.local/api/calibration/neutral-pose/start
# Hold still for 5 seconds
curl -X POST http://belt-node.local/api/calibration/neutral-pose/complete
```

This establishes each node's baseline orientation in the body frame.

### Walk-Through Calibration

Walk your normal walking pace in a straight line for 10 steps.

```bash
curl -X POST http://belt-node.local/api/calibration/walk/start
# Walk 10 steps at normal pace
curl -X POST http://belt-node.local/api/calibration/walk/complete
```

This estimates each node's position on the body relative to the torso reference (belt node IMU).

---

## Step 4: Understanding the Alert System

SENTINEL-WEAR communicates through haptic patterns on the node closest to the approaching direction:

| Alert class | Haptic pattern | Priority |
|---|---|---|
| HumanApproaching slow | Single pulse, 300ms | Info |
| HumanApproaching fast | Double pulse, 100ms each | Warning |
| VehicleNear | Triple pulse with pause | Warning |
| GaitAnomaly (stumble precursor) | Sustained buzz 500ms | Warning |
| FallDetected | Extended buzz 1000ms | Critical |

Direction is encoded by *which node* buzzes: the node closest to the approach direction. An approach from behind right buzzes the right bracelet or right anklet depending on elevation.

---

## Step 5: Companion App

Connect the companion app to the belt node's API:

```
API base: http://belt-node.local  (local Wi-Fi when app and belt are on same network)
         or bluetooth-paired companion app
```

The companion app shows:
- Body-frame radar display (abstract top-down view, no imagery)
- Active tracks with class labels and confidence
- Alert history log
- Battery status per node
- Calibration status

---

## Configuration

```toml
# sentinel-wear.toml

[nodes]
pendant = { sensors = ["MmWaveRadar", "Acoustic", "Imu"], optional_identification = false }
bracelet_left = { sensors = ["MmWaveRadar", "Imu", "Haptic"] }
bracelet_right = { sensors = ["MmWaveRadar", "Imu", "Haptic"] }
belt = { sensors = ["MmWaveRadar", "Imu", "EnvSensor"], role = "PrimaryCompute" }
anklet_left = { sensors = ["LidarTof", "Imu", "Haptic"] }
anklet_right = { sensors = ["LidarTof", "Imu", "Haptic"] }
eyewear = { sensors = ["EventSensor", "Imu"], optional = true }

[tracking]
pentatrack_depth = 2
velocity_method = "Ema"
enable_drift_analysis = true
enable_object_type = true

[alerts]
haptic_enabled = true
audio_enabled = false  # set true if bone-conduction earpiece connected
companion_stream_enabled = true
emergency_contact_enabled = false  # manual trigger only when enabled
```
