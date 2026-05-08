# Getting Started — SENTINEL-WEAR

**A comprehensive guide to setting up, configuring, and operating SENTINEL-WEAR for research, development, and personal use.**

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Architecture Overview](#architecture-overview)
4. [Build and Flash](#build-and-flash)
5. [Simulation Mode](#simulation-mode)
6. [Hardware Setup](#hardware-setup)
7. [Connectivity Configuration](#connectivity-configuration)
8. [Body-Frame Calibration](#body-frame-calibration)
9. [360° Pendant Setup](#360-pendant-setup)
10. [Alert System](#alert-system)
11. [Companion App](#companion-app)
12. [Power Management](#power-management)
13. [Deployment Paths](#deployment-paths)
14. [Configuration Reference](#configuration-reference)
15. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Software

| Requirement | Version | Installation |
|-------------|---------|--------------|
| Rust stable | 1.75+ | `rustup update stable` |
| Embedded target | thumbv7em-none-eabihf | `rustup target add thumbv7em-none-eabihf` |
| Embedded target (aarch64) | aarch64-unknown-linux-gnu | `rustup target add aarch64-unknown-linux-gnu` |
| Git | Latest | `apt install git` or equivalent |

### Hardware Options

**Option A: Simulation Only (No Hardware)**
- Any PC running Linux, macOS, or Windows
- No wearable hardware required
- Suitable for: architecture exploration, algorithm development, testing

**Option B: Physical Deployment**
- SENTINEL-WEAR wearable nodes (pendant, bracelets, belt, anklets, optional eyewear)
- USB-C charger for belt node
- WiFi network (for companion app connection)
- Optional: cellular SIM for remote access

---

## Quick Start

```bash
# Clone and build
git clone https://github.com/<user>/sentinel-wear.git
cd sentinel-wear
cargo build --release

# Run in simulation mode
cargo run --bin sentinel-belt-controller -- \
  --config scenarios/sim_basic.toml \
  --mode simulation

# The simulation runs a synthetic scenario with:
# - Simulated wearer walking at 1.5 m/s
# - Approaching pedestrian (from front-left)
# - Passing vehicle (from rear-right)
# - Gait events and detection alerts
```

**Expected output:**
```
[INFO] sentinel_belt_controller: Initializing simulation mode
[INFO] sentinel_belt_controller: Scenario: sim_basic.toml
[INFO] sentinel_body_frame: Neutral pose calibration: simulated
[INFO] sentinel_tracking: PentaTrack initialized
[INFO] AlertEvent: HumanApproaching bearing=315° range=4.2m velocity=1.2m/s
[INFO] AlertEvent: GaitEvent phase=heel_strike left_anklet
[INFO] AlertEvent: VehicleNear bearing=135° range=8.1m
...
```

---

## Architecture Overview

### The Belt Node as Sole External Gateway

**Critical principle:** Only the belt node connects to external networks (WiFi, Cellular, companion app). All other nodes communicate exclusively via Body-Area Network (BAN) to the belt node.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR Connectivity Model                         │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            ├──► Cellular ──► Remote App     │
│                                            └──► BLE direct ──► Local App   │
│                                                                              │
│  ⚠️  BELT NODE IS THE ONLY NODE WITH WiFi / Cellular / BLE (to external)    │
│  ⚠️  ALL OTHER NODES = BAN ONLY (BLE/UWB to belt, nothing external)         │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why this architecture:**

| Radio | Power | Heat | Form Factor Impact |
|-------|-------|------|-------------------|
| WiFi | 300-1000+ mW | Significant | Breaks jewelry form factor |
| Cellular | 150-1200 mW | Moderate-High | Requires large battery |
| BLE | 5-15 mW | Minimal | Compatible with jewelry |
| UWB | 50-150 mW | Moderate | Compatible with jewelry |

### Transport Layer Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BANDWIDTH HIERARCHY                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BLE 5.x (500 Kbps - 2 Mbps practical)                                       │
│  → Control plane (configuration, commands, health)                          │
│  → Metadata (detections, IMU orientation, classification results)            │
│  → Low-bandwidth sensor data                                                 │
│  → Always-on, lowest power (5-15 mW)                                         │
│  → Present on ALL nodes                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  UWB (4-6 Mbps sustained, 6.8 Mbps peak)                                     │
│  → Precision time synchronization (sub-nanosecond)                          │
│  → Precision ranging (cm-level distance between nodes)                       │
│  → Moderate-bandwidth continuous (single camera at 720p-1080p)               │
│  → High-bandwidth BURSTS (compressed clips, short recordings)                │
│  → 360° at 2K-2.5K resolution                                                │
│  → All 8 cameras at QVGA-VGA simultaneously                                  │
│  → Optional on nodes, enabled per configuration                              │
│  → Power: 50-150 mW when active                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  WiFi (50-300+ Mbps) — BELT NODE ONLY                                        │
│  → High-bandwidth continuous (4K 360° live streaming)                        │
│  → Multiple simultaneous camera streams                                      │
│  → SLAM data transfer                                                        │
│  → Primary path to companion app                                             │
│  → Power: 300-500 mW when streaming                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Cellular (Variable by module) — BELT NODE ONLY                              │
│  → LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips                │
│  → LTE Cat 4: 50-150 Mbps — good for moderate streaming                     │
│  → 5G: 100-1000+ Mbps — suitable for 360° live relay                         │
│  → Used when WiFi unavailable (remote access, away from home)               │
│  → Power: 150-1200 mW depending on module and activity                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Build and Flash

### Building the Main Belt Controller

```bash
# Clone repository
git clone https://github.com/<user>/sentinel-wear.git
cd sentinel-wear

# Build all workspace crates
cargo build --release --workspace

# Run tests
cargo test --workspace

# The main binary is produced at:
# target/release/sentinel-belt-controller
```

### Building Firmware for Embedded Nodes

**Standard pendant node:**
```bash
cd firmware
cargo build --release --target thumbv7em-none-eabihf --bin pendant_node
# Output: target/thumbv7em-none-eabihf/release/pendant_node
```

**360° pendant node:**
```bash
cargo build --release --target thumbv7em-none-eabihf --bin pendant_360 --features curved_360
```

**Bracelet nodes:**
```bash
cargo build --release --target thumbv7em-none-eabihf --bin bracelet_left
cargo build --release --target thumbv7em-none-eabihf --bin bracelet_right
```

**Anklet nodes:**
```bash
cargo build --release --target thumbv7em-none-eabihf --bin anklet_left
cargo build --release --target thumbv7em-none-eabihf --bin anklet_right
```

**Eyewear node:**
```bash
cargo build --release --target thumbv7em-none-eabihf --bin eyewear_node
```

**Belt node (MCU variant):**
```bash
cargo build --release --target thumbv7em-none-eabihf --bin belt_node
```

**Belt node (Linux SoM variant):**
```bash
cargo build --release --target aarch64-unknown-linux-gnu --bin belt_node
```

### Flashing Firmware

**Using probe-rs:**
```bash
# Flash pendant node
probe-rs download target/thumbv7em-none-eabihf/release/pendant_node \
  --chip STM32H743VITx \
  --protocol SWD

# Flash bracelet node
probe-rs download target/thumbv7em-none-eabihf/release/bracelet_left \
  --chip nRF5340 \
  --protocol SWD
```

**Using OpenOCD:**
```bash
openocd -f interface/stlink.cfg -f target/stm32h7x.cfg \
  -c "program target/thumbv7em-none-eabihf/release/pendant_node verify reset exit"
```

---

## Simulation Mode

### Running Without Hardware

Simulation mode allows full system testing without physical nodes:

```bash
cargo run --bin sentinel-belt-controller -- \
  --config scenarios/sim_basic.toml \
  --mode simulation
```

### Available Scenarios

| Scenario | Description |
|----------|-------------|
| `sim_basic.toml` | Basic walking with approaching pedestrian and vehicle |
| `sim_gait.toml` | Focus on gait events and stumble detection |
| `sim_extreme_velocity.toml` | Projectile detection simulation |
| `sim_360_pendant.toml` | 360° pendant operation with camera simulation |
| `sim_multi_room.toml` | Multi-environment traversal with SLAM |

### Creating Custom Scenarios

```toml
# scenarios/my_scenario.toml

[scenario]
name = "my_custom_scenario"
duration_s = 60

[wearer]
initial_position = [0.0, 0.0, 0.0]  # x, y, z in meters
walking_speed_ms = 1.5
walking_pattern = "normal"  # "normal" | "fast" | "irregular"

[environment]
rooms = [
  { name = "living_room", bounds = [[0, 5], [0, 4]] },
  { name = "kitchen", bounds = [[5, 8], [0, 4]] }
]

[[objects]]
type = "approaching_human"
initial_position = [10.0, 0.0, 0.0]
velocity_ms = 1.2
approach_angle_deg = 45  # From front-right

[[objects]]
type = "passing_vehicle"
initial_position = [-5.0, 8.0, 0.0]
velocity_ms = 8.0
approach_angle_deg = 135  # From rear-right

[extreme_velocity]
enabled = false
projectile_type = "rifle"  # "handgun" | "rifle" | "fragmentation"
projectile_velocity_ms = 850
approach_angle_deg = 0  # Directly from front
```

### Simulation Outputs

The simulator produces structured logs and can export recordings:

```bash
# Run with recording export
cargo run --bin sentinel-belt-controller -- \
  --config scenarios/sim_basic.toml \
  --mode simulation \
  --export-recordings ./simulation_output/

# Output structure:
# ./simulation_output/
# ├── detections.jsonl
# ├── tracks.jsonl
# ├── alerts.jsonl
# ├── gait_events.jsonl
# └── world_model.json
```

---

## Hardware Setup

### Node Inventory

**Minimum viable system (Path 1 - Basic Awareness):**

| Node | Quantity | Notes |
|------|----------|-------|
| Belt Node | 1 | MCU variant (Variant A) or ESP32 variant (Variant D) |
| Pendant | 1 | Standard variant (Variant A), no camera |
| Bracelet | 2 | Minimal configuration |
| Anklet | 2 | Minimal configuration |

**Standard system (Path 2 - Personal Awareness):**

| Node | Quantity | Notes |
|------|----------|-------|
| Belt Node | 1 | Linux SoM variant (Variant B) |
| Pendant | 1 | Standard variant with camera |
| Bracelet | 2 | Standard configuration |
| Anklet | 2 | Extended configuration |
| Eyewear | 1 (optional) | Event-only variant (Variant A) |

**Full capability system (Path 3 - Security Professional):**

| Node | Quantity | Notes |
|------|----------|-------|
| Belt Node | 1 | Linux SoM variant (Variant B or C) |
| Pendant | 1 | 360° Curved variant (Variant B) |
| Bracelet | 2 | Extended configuration with camera |
| Anklet | 2 | Extended configuration |
| Eyewear | 1 | Full array variant (Variant C) |

### Charging

| Node | Charging Method | Charge Time |
|------|-----------------|--------------|
| Pendant Standard | USB-C or magnetic pogo | 1-2 hours |
| Pendant 360° | USB-C or belt power input | 2-4 hours |
| Bracelet | USB-C or magnetic pogo | 30-60 minutes |
| Anklet | USB-C or Qi wireless | 1-2 hours |
| Eyewear (clip-on) | USB-C | 30-60 minutes |
| Belt Node | USB-C PD | 3-5 hours |
| Belt Node (hot-swap) | USB-C PD + battery swap | Swap < 10 seconds |

### Power-On Sequence

1. **Belt node first** — Power on belt node, wait for boot (5-15 seconds depending on variant)
2. **Other nodes in any order** — Pendant, bracelets, anklets, eyewear
3. **Automatic discovery** — Nodes appear on BAN within 1-2 seconds of power-on
4. **Calibration check** — System verifies calibration status of all nodes

### Physical Placement

| Node | Position | Orientation Notes |
|------|----------|-------------------|
| Pendant | Chest/collar level | Hangs naturally; face outward |
| Pendant 360° | Chest/collar level | Camera arc faces outward |
| Bracelet Left | Left wrist/forearm | Sensor window faces outward (away from body) |
| Bracelet Right | Right wrist/forearm | Sensor window faces outward |
| Belt Node | Waist | Comfortably snug; buckle centered |
| Anklet Left | Above left ankle | Sensor faces forward-downward |
| Anklet Right | Above right ankle | Sensor faces forward-downward |
| Eyewear | On glasses or headband | Event camera faces forward |

---

## Connectivity Configuration

### WiFi Setup (Belt Node Only)

The belt node connects to your home WiFi network. This is the primary connection for the companion app.

**Configuration:**

```toml
[connectivity]
wifi_enabled = true
wifi_ssid = "YourHomeNetwork"
wifi_password = ""  # Set at runtime for security: via companion app or CLI
wifi_prefer_5ghz = true  # Strongly recommended for RF coexistence
```

**Runtime setup via CLI:**
```bash
# Set WiFi credentials (stored in secure storage on belt node)
curl -X POST http://belt-node.local/api/connectivity/wifi \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ssid": "YourHomeNetwork", "password": "YourPassword"}'
```

### Cellular/SIM Setup (Belt Node Only, Optional)

For remote access when away from WiFi:

**Physical setup:**
1. Insert nano-SIM into belt node SIM slot
2. Ensure cellular module is installed (Quectel EC21/EC25/RM502Q or eSIM variant)

**Configuration:**

```toml
[connectivity.cellular]
enabled = true
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"         # APN from your carrier
fallback_to_wifi = true          # Prefer WiFi when available
always_on_cellular = false       # Keep cellular active even with WiFi
alert_via_cellular = true        # Send critical alerts via cellular
stream_via_cellular = false      # Stream video over cellular (data cost)
emergency_contact = ""           # Phone/endpoint for emergency alerts
```

**Supported cellular modules:**

| Module | Technology | Speed | Interface | Use Case |
|--------|------------|-------|-----------|----------|
| Quectel EC21 | LTE Cat 1 | 5-10 Mbps | UART | Alerts, metadata only |
| Quectel EC25 | LTE Cat 4 | 50-150 Mbps | UART/USB | Compressed video clips |
| Quectel RM502Q-AE | 5G Sub-6GHz | 100+ Mbps | USB 3.x | Live 360° streaming |
| Quectel EG912Y-GL | LTE Cat 12 | 300+ Mbps | USB | Multi-carrier, eSIM |

**Checking cellular status:**
```bash
curl http://belt-node.local/api/connectivity/cellular

# Response:
{
  "enabled": true,
  "connected": true,
  "carrier": "T-Mobile",
  "technology": "LTE",
  "rssi_dbm": -72,
  "signal_quality": "good",
  "data_used_mb": 42,
  "module": "Quectel EC25"
}
```

### BAN Configuration (All Nodes)

**BLE scheduling:**

```toml
[connectivity.ban]
primary = "ble"                  # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30

[connectivity.ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15          # Higher priority for gait detection
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100        # Optional node, lower priority
sleep_interval_ms = 500          # When all nodes in idle mode
```

**Connection interval tradeoffs:**

| Interval | Latency | Power per Node | Use Case |
|----------|---------|----------------|----------|
| 7.5 ms | ~5 ms | 15-20 mW | Real-time motion capture |
| 15 ms | ~10 ms | 10-15 mW | Active gait monitoring |
| 30 ms | ~20 ms | 5-10 mW | Standard operation (default) |
| 100 ms | ~60 ms | 2-5 mW | Power saving mode |
| 1000 ms | ~600 ms | <1 mW | Sleep mode |

**UWB configuration (optional):**

```toml
[connectivity.ban.uwb]
enabled = false
role = "timing_and_ranging"     # "timing_only" | "timing_and_ranging" | "full_bandwidth"
streaming_fallback = false      # Use UWB for camera streaming when WiFi unavailable
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]
```

**UWB roles:**

| Role | Capabilities | Power | Use Case |
|------|-------------|-------|----------|
| `timing_only` | Sub-ns sync, cm ranging | ~50 mW | Precision timing |
| `timing_and_ranging` | Above + occasional bursts | ~80 mW | Enhanced body-frame |
| `full_bandwidth` | Above + 5-6 Mbps sustained | 100-150 mW | Camera streaming, 360° |

### RF Coexistence

**Recommended configuration:**

```toml
[rf_coexistence]
wifi_prefer_5ghz = true        # Eliminates 2.4 GHz conflict with BLE
ble_wifi_timeshare = false     # Not needed if 5 GHz WiFi used
ble_gap_during_wifi_ms = 5     # If 2.4 GHz WiFi required
```

---

## Body-Frame Calibration

Calibration establishes the geometric relationship between each node and the body-frame origin (belt IMU). Two calibrations are required:

### Neutral-Pose Calibration

Establishes each node's baseline orientation.

**Procedure:**
1. Put on all nodes at their intended positions
2. Stand in neutral anatomical pose:
   - Feet hip-width apart, weight balanced
   - Arms hanging naturally at sides
   - Head facing forward, level
   - Torso upright

3. Hold for 5 seconds without movement

```bash
# Start calibration
curl -X POST http://belt-node.local/api/calibration/neutral-pose/start

# System records for 5 seconds

# Check status
curl http://belt-node.local/api/calibration/neutral-pose/status
# Response: {"status": "Complete", "confidence": 0.97}
```

**What happens:**
- Each node's IMU Madgwick filter state is recorded
- Relative quaternion between each node and belt IMU is stored
- This becomes the "mounting transform" for each node

### Walk-Through Calibration

Estimates each node's position on the body relative to the belt.

**Procedure:**
1. After neutral-pose calibration completes
2. Walk at normal pace for 10-20 steps
3. Include natural arm swing
4. Occasional direction changes are acceptable

```bash
# Start walk calibration
curl -X POST http://belt-node.local/api/calibration/walk/start

# Walk 10-20 steps

# Complete calibration
curl -X POST http://belt-node.local/api/calibration/walk/complete

# Check results
curl http://belt-node.local/api/calibration/walk/status
```

**Expected response:**
```json
{
  "status": "Complete",
  "nodes": {
    "pendant": {"confidence": 0.94, "position_m": [0.0, 0.15, 0.4]},
    "bracelet_left": {"confidence": 0.91, "position_m": [-0.35, 0.05, 0.8]},
    "bracelet_right": {"confidence": 0.92, "position_m": [0.35, 0.05, 0.8]},
    "anklet_left": {"confidence": 0.88, "position_m": [-0.12, 0.0, 0.05]},
    "anklet_right": {"confidence": 0.89, "position_m": [0.12, 0.0, 0.05]}
  },
  "overall_confidence": 0.91
}
```

**Confidence thresholds:**

| Confidence | Assessment | Action |
|------------|------------|--------|
| > 0.85 | Excellent | Proceed with normal operation |
| 0.65-0.85 | Acceptable | Re-calibrate if alerts feel imprecise |
| < 0.65 | Poor | Re-calibrate immediately |

### When to Re-Calibrate

- After repositioning any node
- After replacing a node with a new unit
- If directional alerts feel significantly wrong (>30° off)
- After extended storage (>1 month)
- If calibration confidence drops during operation

### Calibration for 360° Pendant

The 360° pendant requires additional camera calibration:

**Camera intrinsic calibration:**
```bash
# Print calibration board
sentinel-wear-cli print-calibration-board --type aruco_4x4 --size a4

# Run calibration (hold board at 1m, 2m, 3m at various angles)
sentinel-wear-cli calibrate-cameras --node pendant_360 --frames 40 --output pendant_cal.json
```

**Camera extrinsic calibration:**
```bash
# Establish relative positions of cameras in the arc
sentinel-wear-cli calibrate-camera-extrinsics --node pendant_360 --output extrinsics.json

# Verify stitching quality
sentinel-wear-cli verify-stitching --node pendant_360 --calibration pendant_cal.json
```

**Quality check:** RMS reprojection error should be < 0.8 pixels per camera. Values > 2.0 indicate poor calibration.

---

## 360° Pendant Setup

### Architecture Overview

The 360° curved pendant uses a wired internal camera bus architecture:

```
8 Cameras → Internal MIPI Bus → Vision Processor on Pendant
                                        │
                                        ├── Stitch on-pendant (optional)
                                        ├── Encode (H.264/H.265)
                                        └── Stream to Belt via BLE/UWB
                                                │
                                                ▼
                                           Belt Node
                                                │
                                                ▼
                                        WiFi/Cellular → Companion App
```

### Configuration Options

**Camera count:**
```toml
[nodes.pendant_360]
camera_count = 8              # 4 | 6 | 8 | 12
camera_resolution = "720p"     # "qvga" | "vga" | "720p" | "1080p"
```

**Stitching location:**
```toml
[nodes.pendant_360.stitching]
location = "pendant"          # "pendant" | "belt"
quality = "high"              # "low" | "medium" | "high"
fps = 30
```

**Progressive quality (for bandwidth-limited links):**
```toml
[nodes.pendant_360.progressive_quality]
enabled = true
baseline_resolution = "qvga"
refinement_resolution = "720p"
refinement_interval_ms = 500
adaptive_bandwidth = true
```

### Bandwidth Over UWB

| Configuration | Bandwidth | UWB Capability |
|---------------|-----------|----------------|
| 8 cameras × QVGA MJPEG | ~0.5-1 Mbps | ✅ Easily handled |
| 8 cameras × VGA H.264 | ~2-4 Mbps | ✅ Handled |
| Stitched 360° at 2K H.264 | ~3-4 Mbps | ✅ Handled |
| Stitched 360° at 2.5K H.264 | ~5-6 Mbps | ⚠️ At maximum |
| Stitched 360° at 4K H.264 | ~8-15 Mbps | ❌ Requires WiFi |

### Extended Runtime with Belt Power

For all-day 360° operation, the pendant can draw power from the belt:

```toml
[nodes.pendant_360.power]
belt_power_input = true       # Draw from belt battery via conductive chain
```

This eliminates pendant battery constraints but requires:
- Conductive necklace chain or separate power cable
- Belt node with high-capacity battery (Variant E recommended)

---

## Alert System

### Haptic Alert Patterns

SENTINEL-WEAR communicates threat and status information through directional haptic patterns:

| Alert Class | Pattern | Duration | Priority | Nodes Involved |
|-------------|---------|----------|----------|-----------------|
| HumanApproaching slow | Single pulse | 300 ms | Info | Nearest to approach direction |
| HumanApproaching fast | Double pulse | 100+100 ms | Warning | Nearest to approach direction |
| VehicleNear | Triple pulse | 100+100+100 ms | Warning | Pendant + nearest bracelet |
| GaitAnomaly | Sustained buzz | 500 ms | Warning | Anklets |
| StumblePrecursor | Rapid pulses | 50 ms × 5 | Warning | Anklets |
| FallDetected | Extended buzz | 1000 ms | Critical | All nodes |
| FastObjectDetected | Rapid pulses | 50 ms × 5 | Critical | Pendant, eyewear |
| CalibrationPrompt | Slow sweep | 100 ms × 3 | Info | Pendant |

### Directional Encoding

The node that vibrates indicates the direction of the detected event:

```
                 Front
                   ▲
                   │
         Pendant ──┼── Pendant
                   │
    Left Bracelet ─┼─ Right Bracelet
                   │
    Left Anklet ───┼─── Right Anklet
                   │
```

**Example:** An approach from behind-right vibrates:
1. Right bracelet (if elevated source)
2. Right anklet (if ground-level source)

**Multi-node alert:** For critical events (fall detected, extreme velocity), all nodes vibrate to maximize attention.

### QoS Classes for Alert Priority

Alerts are transmitted using QoS classes that determine transmission priority:

| QoS Class | Priority | Traffic Types | Latency Target |
|-----------|----------|---------------|----------------|
| Critical | Highest | Extreme velocity, fall detection, emergency | < 5 ms |
| High | Second | Gait anomaly, stumble precursor | < 20 ms |
| Medium | Third | Presence detection, tracking | < 50 ms |
| Low | Fourth | Battery status, telemetry | < 500 ms |

**Critical QoS behavior:**
- Preempts other BLE traffic
- Transmitted immediately (doesn't wait for scheduled slot)
- Guaranteed reserved slot every cycle

### Audio Alerts (Optional)

If bone-conduction earpiece connected:

```toml
[alerts.audio]
enabled = false                # Set true if earpiece connected
device = "bone_conduction"     # "bone_conduction" | "speaker"
```

| Alert | Audio Cue |
|-------|-----------|
| HumanApproaching | Tone with bearing-encoded pitch (higher = more right) |
| VehicleNear | Distinct vehicle-class tone |
| FallDetected | Persistent tone until acknowledged |
| BatteryLow | Battery warning pattern |

---

## Companion App

### Connection Modes

| Mode | Connection | Bandwidth | Use Case |
|------|------------|-----------|----------|
| Primary | WiFi (same network) | High | Full features |
| Remote | Cellular | Variable | Alerts, metadata, limited streaming |
| Fallback | BLE direct | Low | Alerts, configuration only |

### Connecting

**WiFi (primary):**
1. Ensure phone/computer is on same WiFi network as belt node
2. Open companion app
3. App auto-discovers belt node via mDNS (`belt-node.local`)
4. If not discovered, enter belt node IP manually
5. Enter bearer token (set in `sentinel-wear.toml`)

**Cellular (remote):**
1. Enable remote access in configuration:
   ```toml
   [companion_app]
   enable_remote_access = true
   remote_access_token = "your-secure-token-here"
   ```
2. In app, select "Remote Connection"
3. Enter belt node's cellular IP or dynamic DNS hostname
4. Enter remote access token

### App Features

**Live Awareness View:**
- Body-centric radar display
- Detected entities as blobs with velocity vectors
- PentaTrack prediction arcs
- Alert zones as colored rings
- Entity classification labels with confidence

**360° Live View (360° pendant only):**
- Equirectangular panorama
- Interactive spherical view
- VR headset support

**3D World Model (Linux SoM belt only):**
- Dense SLAM reconstruction
- Entity trajectories overlaid
- Time-scrubbing to replay past states

**Recordings:**
- Full library of stored recordings
- Filter by node, date, trigger type, classification
- Playback with timeline scrubbing
- Download and export

**Analytics:**
- Gait trends over time
- Detection heatmaps
- Stride asymmetry charts
- Alert frequency analysis

**Configuration:**
- Node management
- Policy mode switching
- Data handling settings
- Calibration tools

### Legal Export

For evidence or personal records:

1. Select recording in app
2. Tap "Export"
3. Choose format: ZIP with integrity manifest
4. Export includes:
   - Recording file
   - SHA-256 hash
   - Device ID and firmware version
   - Timestamp chain
   - Export timestamp

---

## Power Management

### Power Profiles

SENTINEL-WEAR supports multiple power profiles to balance capability and runtime:

```toml
[power.profile]
mode = "standard"              # "minimal" | "standard" | "full_active" | "custom"
```

| Profile | Active Features | Belt Runtime (5000 mAh) |
|---------|-----------------|-------------------------|
| minimal | Sparse tracking, BLE only, no streaming | 20-30 hours |
| standard | Sparse tracking, WiFi idle, occasional streaming | 12-15 hours |
| full_active | Dense SLAM, 360° streaming, all features | 5-8 hours |

### Adaptive Power Management

The system can automatically adjust features based on battery level:

```toml
[power.adaptive]
enabled = true

[power.adaptive.thresholds]
# At 50% battery: reduce features
level_50 = ["disable_slam", "reduce_camera_fps"]

# At 25% battery: essential features only
level_25 = ["sparse_only", "disable_streaming"]

# At 10% battery: emergency mode
level_10 = ["alerts_only", "disable_all_cameras"]
```

### Node Sleep Management

Nodes can enter low-power sleep when idle:

```toml
[power.sleep]
idle_timeout_minutes = 5        # Sleep after 5 minutes of no detection
wake_on_motion = true           # PIR or radar can wake from sleep
connection_interval_sleep_ms = 1000  # Slow BLE interval while sleeping
```

### Battery Monitoring

Check battery status for all nodes:

```bash
curl http://belt-node.local/api/nodes/battery

# Response:
{
  "pendant": {"percent": 78, "voltage_mv": 3950, "charging": false},
  "bracelet_left": {"percent": 92, "voltage_mv": 4050, "charging": false},
  "bracelet_right": {"percent": 89, "voltage_mv": 4000, "charging": false},
  "anklet_left": {"percent": 85, "voltage_mv": 3980, "charging": false},
  "anklet_right": {"percent": 87, "voltage_mv": 3990, "charging": false},
  "belt": {"percent": 65, "voltage_mv": 3700, "charging": true}
}
```

### Thermal Management

The belt node monitors temperature and throttles if needed:

```toml
[thermal]
monitoring_enabled = true
warning_threshold_c = 40
throttle_threshold_c = 45
shutdown_threshold_c = 50
throttle_action = "reduce_slam_rate"  # "reduce_slam_rate" | "reduce_camera_fps" | "disable_slam"
```

When thermal limits are reached, the companion app displays a warning and reduced performance is expected.

---

## Deployment Paths

Different use cases warrant different node configurations. Below are recommended paths:

### Path 1: Basic Awareness (Minimal Cost)

**Target:** Casual user wanting basic presence awareness, minimal investment

| Node | Variant/Config |
|------|----------------|
| Belt | Variant A (MCU) or D (ESP32) |
| Pendant | Variant A, no camera |
| Bracelets | Minimal config, no camera |
| Anklets | Minimal config |

**Expected cost:** Lowest
**Runtime:** 12-20 hours
**Capabilities:** Sparse tracking, haptic alerts, no streaming

### Path 2: Standard Personal Awareness

**Target:** Consumer wanting full awareness with streaming capability

| Node | Variant/Config |
|------|----------------|
| Belt | Variant B (Linux SoM) |
| Pendant | Variant A with camera |
| Bracelets | Standard config |
| Anklets | Extended config |
| Eyewear | Optional, Variant A |

**Runtime:** 10-15 hours (sparse), 5-8 hours (active)
**Capabilities:** Sparse + dense SLAM, streaming, recording

### Path 3: Security Professional

**Target:** Security personnel, journalists requiring 360° awareness

| Node | Variant/Config |
|------|----------------|
| Belt | Variant B or C (Linux SoM) |
| Pendant | Variant B (360° Curved) |
| Bracelets | Extended with camera |
| Anklets | Extended config |
| Eyewear | Variant B or C |

**Runtime:** 5-8 hours (full active)
**Capabilities:** Full 360° awareness, dense SLAM, legal evidence capture

### Path 4: Extended Runtime / Continuous Use

**Target:** Users requiring 24/7 operation

| Node | Variant/Config |
|------|----------------|
| Belt | Variant E (Hot-swappable battery) |
| Pendant | Variant D (Tactical) or B (360°) with belt power |
| Bracelets | Standard config |
| Anklets | Standard config |

**Runtime:** Unlimited with battery swaps
**Capabilities:** Full capability with no downtime

### Path 5: Extreme Velocity Detection

**Target:** Users in environments with potential for high-speed threats

| Node | Variant/Config |
|------|----------------|
| Belt | Variant B or C |
| Pendant | Variant E (Event-Enhanced) |
| Bracelets | Standard config |
| Anklets | Standard config |
| Eyewear | Variant A (Event-only) |

**Runtime:** 8-12 hours
**Capabilities:** Full awareness + projectile detection

### Path 6: Accessibility (Visually Impaired)

**Target:** Visually impaired users wanting haptic spatial awareness

| Node | Variant/Config |
|------|----------------|
| Belt | Variant B (Linux SoM) |
| Pendant | Variant B (360° Curved) |
| Bracelets | Standard config (directional haptics essential) |
| Anklets | Standard config |

**Runtime:** 10-15 hours
**Capabilities:** Haptic directional alerts, audio scene description, obstacle detection

---

## Configuration Reference

### Complete Configuration File

```toml
# sentinel-wear.toml - Complete Configuration Reference

# ============================================
# NODE DEFINITIONS
# ============================================

[nodes]
# Pendant - Standard variant
pendant = { 
  sensors = ["MmWaveRadar", "Acoustic", "Imu"], 
  variant = "standard",
  optional_camera = true,
  camera_enabled = false
}

# Pendant - 360° Curved variant (if used)
pendant_360 = { 
  sensors = ["MmWaveRadar", "Acoustic", "Imu", "Camera360"], 
  variant = "curved",
  camera_count = 8,
  camera_resolution = "720p",
  enabled = false
}

# Bracelets
bracelet_left = { 
  sensors = ["MmWaveRadar", "Imu", "Haptic"], 
  variant = "standard",
  optional_camera = false
}
bracelet_right = { 
  sensors = ["MmWaveRadar", "Imu", "Haptic"], 
  variant = "standard",
  optional_camera = false
}

# Belt node
belt = { 
  sensors = ["MmWaveRadar", "Imu", "EnvSensor"], 
  variant = "linux_som",
  role = "PrimaryCompute"
}

# Anklets
anklet_left = { 
  sensors = ["LidarTof", "Imu", "Haptic"], 
  variant = "standard"
}
anklet_right = { 
  sensors = ["LidarTof", "Imu", "Haptic"], 
  variant = "standard"
}

# Eyewear (optional)
eyewear = { 
  sensors = ["EventSensor", "Imu"], 
  variant = "clip_on",
  enabled = false
}

# ============================================
# CONNECTIVITY
# ============================================

[connectivity]
# WiFi - Primary connection to companion app
wifi_enabled = true
wifi_prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false
emergency_contact = ""

[connectivity.bluetooth]
ble_direct_enabled = true
ble_advertising_name = "sentinel-wear"

[connectivity.ban]
primary = "ble"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

[connectivity.ban.ble_scheduling]
default_connection_interval_ms = 30
anklet_interval_ms = 15
pendant_interval_ms = 30
bracelet_interval_ms = 50
eyewear_interval_ms = 100
sleep_interval_ms = 500

[connectivity.ban.uwb]
enabled = false
role = "timing_and_ranging"
streaming_fallback = false
max_sustained_mbps = 5
nodes_with_uwb = ["pendant", "anklet_left", "anklet_right"]

[connectivity.ban.antenna_diversity]
enabled = true
antenna_count = 2
selection_mode = "rssi"
switch_threshold_db = 5
preferred_antenna = "auto"

[connectivity.ban.qos]
classes = ["critical", "high", "medium", "low"]

[connectivity.ban.qos.critical]
preempt_other_traffic = true
max_latency_ms = 5
traffic_types = ["extreme_velocity", "fall_detection", "emergency_alert"]

[connectivity.ban.qos.high]
priority_boost = 1.5
max_latency_ms = 20
traffic_types = ["gait_anomaly", "stumble_precursor"]

[connectivity.ban.qos.medium]
max_latency_ms = 50
traffic_types = ["presence_detection", "tracking_update"]

[connectivity.ban.qos.low]
max_latency_ms = 500
traffic_types = ["battery_status", "health_telemetry"]

# RF Coexistence
[rf_coexistence]
wifi_prefer_5ghz = true
ble_wifi_timeshare = false
ble_gap_during_wifi_ms = 5

# ============================================
# TRACKING AND PERCEPTION
# ============================================

[tracking]
pentatrack_depth = 2
velocity_method = "Ema"
enable_drift_analysis = true
enable_object_type = true
prediction_horizon_s = 2.0

[perception]
# Radar pipeline
radar_cfard_threshold = 3.0
radar_doppler_bins = 64

# Acoustic pipeline
acoustic_sample_rate_hz = 16000
acoustic_fft_size = 512
acoustic_doa_enabled = true

# IMU fusion
imu_filter = "madgwick"
imu_sample_rate_hz = 200

# ============================================
# ALERTS
# ============================================

[alerts]
haptic_enabled = true
audio_enabled = false
companion_stream_enabled = true
emergency_contact_enabled = false

[alerts.haptic]
intensity_default = 70        # 0-100
intensity_critical = 100
duration_ms_default = 300

[alerts.extreme_velocity]
enabled = false
mode = "disabled"             # "disabled" | "research" | "production"
min_velocity_ms = 50
max_detection_distance_m = 10
alert_on_detection = true
preempt_all_traffic = true

# ============================================
# DATA HANDLING
# ============================================

[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "sd_card"
retention_days = 30
continuous_recording = false
recording_trigger = "on_detection"
max_storage_mb = 0
include_integrity_chain = true

[data.360_pendant]
store_panorama = true
panorama_resolution = "2k"
panorama_fps = 15
keyframe_interval_s = 2.0

# ============================================
# PRIVACY
# ============================================

[privacy.camera]
hardware_switch_installed = false
software_control_enabled = true
default_state = "disabled"
schedule = []                  # Empty = always disabled; or [["08:00", "22:00"]]
trigger_on_detection = false

# ============================================
# POWER MANAGEMENT
# ============================================

[power.profile]
mode = "standard"

[power.adaptive]
enabled = true

[power.adaptive.thresholds]
level_50 = ["disable_slam", "reduce_camera_fps"]
level_25 = ["sparse_only", "disable_streaming"]
level_10 = ["alerts_only", "disable_all_cameras"]

[power.sleep]
idle_timeout_minutes = 5
wake_on_motion = true
connection_interval_sleep_ms = 1000

# ============================================
# THERMAL MANAGEMENT
# ============================================

[thermal]
monitoring_enabled = true
warning_threshold_c = 40
throttle_threshold_c = 45
shutdown_threshold_c = 50
throttle_action = "reduce_slam_rate"

# ============================================
# COMPANION APP
# ============================================

[companion_app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9091
enable_remote_access = false
remote_access_token = ""

# ============================================
# CALIBRATION
# ============================================

[calibration]
auto_calibrate_on_boot = false
confidence_threshold = 0.85
walk_through_steps = 15

# ============================================
# WORLD MODEL
# ============================================

[world_model]
mode = "dual"                  # "sparse" | "dense" | "dual"

[world_model.sparse]
always_enabled = true
update_rate_hz = 20

[world_model.dense]
enabled = true
slam_backend = "lio_sam"       # "lio_sam" | "fast_lio2" | "orb_slam3"
map_resolution_m = 0.05
loop_closure_enabled = true
```

---

## Troubleshooting

### Common Issues

**Node not discovered on BAN:**
1. Check node is powered on (LED status)
2. Verify node is within 2 m of belt
3. Check BAN configuration matches between node and belt
4. Restart belt node BAN service:
   ```bash
   curl -X POST http://belt-node.local/api/ban/restart
   ```

**Calibration confidence low:**
1. Ensure all nodes are properly positioned
2. Redo neutral-pose calibration, holding still for full 5 seconds
3. For walk-through: walk at normal pace, avoid stopping mid-calibration
4. Check for IMU errors:
   ```bash
   curl http://belt-node.local/api/nodes/{node_id}/health
   ```

**WiFi not connecting:**
1. Verify SSID and password are correct
2. Check WiFi band preference:
   ```bash
   curl http://belt-node.local/api/connectivity/wifi
   ```
3. Try 2.4 GHz if 5 GHz unavailable:
   ```toml
   [connectivity]
   wifi_prefer_5ghz = false
   ```

**Cellular not registering:**
1. Check SIM is inserted correctly
2. Verify APN matches carrier:
   ```bash
   curl http://belt-node.local/api/connectivity/cellular
   ```
3. Try manual network selection:
   ```bash
   curl -X POST http://belt-node.local/api/connectivity/cellular/register \
     -d '{"carrier": "T-Mobile"}'
   ```

**360° pendant stitching artifacts:**
1. Verify FSYNC is configured
2. Check camera calibration:
   ```bash
   sentinel-wear-cli verify-stitching --node pendant_360
   ```
3. Re-calibrate if RMS error > 1.0 pixels

**High latency alerts:**
1. Check BLE scheduling configuration
2. Verify QoS is set for critical alerts:
   ```bash
   curl http://belt-node.local/api/ban/qos
   ```
3. Enable reserved slot for critical alerts:
   ```toml
   [connectivity.ban.qos.critical]
   preempt_other_traffic = true
   ```

**Battery draining fast:**
1. Check power profile:
   ```bash
   curl http://belt-node.local/api/power/profile
   ```
2. Enable adaptive power management
3. Reduce camera FPS or resolution
4. Check for thermal throttling (reduced performance)

**Thermal warnings:**
1. Check current temperature:
   ```bash
   curl http://belt-node.local/api/thermal
   ```
2. Reduce compute load (disable SLAM, reduce camera FPS)
3. Ensure enclosure ventilation is not blocked

### Debug Logging

Enable verbose logging:

```bash
curl -X POST http://belt-node.local/api/debug \
  -d '{"level": "debug", "modules": ["ban", "tracking", "alerts"]}'
```

View logs:
```bash
curl http://belt-node.local/api/logs?lines=100
```

### Reset to Defaults

```bash
curl -X POST http://belt-node.local/api/config/reset
```

### Support Resources

- GitHub Issues: `https://github.com/<user>/sentinel-wear/issues`
- Documentation: `docs/` directory in repository
- Community discussions: GitHub Discussions

---

## Next Steps

After completing this guide:

1. **Explore the companion app** — Connect and familiarize yourself with all views
2. **Review theory documentation** — Understand the architecture in depth:
   - `docs/theory/body_coordinate_fusion.md`
   - `docs/theory/slam_world_model.md`
   - `docs/theory/extreme_velocity_sensing.md`
3. **Customize configuration** — Adjust `sentinel-wear.toml` for your specific needs
4. **Join the community** — Share experiences and get help

---

**End of Getting Started Guide**
