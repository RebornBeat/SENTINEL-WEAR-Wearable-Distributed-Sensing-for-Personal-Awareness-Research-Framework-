# Test Jig PCB — SENTINEL-WEAR Production Testing

**Project:** SENTINEL-WEAR
**Purpose:** Automated validation for all SENTINEL-WEAR node types and all variants.
**Status:** Reference Design v2.0

---

## 1. Overview

The SENTINEL-WEAR Universal Test Station (UTS) validates all wearable node variants before deployment. It covers:

- **Electrical validation:** Power rails, current draw, battery charging
- **Functional sensor testing:** All sensing modalities including extreme velocity detection
- **Data handling verification:** All storage and streaming modes
- **Privacy control testing:** Hardware switch and software configuration
- **Connectivity testing:** BLE, UWB, WiFi, Cellular (belt node only)
- **Performance validation:** Latency, bandwidth, RF coexistence
- **Environmental testing:** IP67, thermal management
- **Firmware verification:** Build hash, calibration data, configuration

**All node variants are tested through the same carrier board by swapping node-specific test shields.**

---

## 2. Architectural Principle: Belt Node as Sole External Gateway

**Critical test distinction:**

| Node Type | Has WiFi | Has Cellular | Test Approach |
|-----------|----------|--------------|---------------|
| Pendant | ❌ No | ❌ No | BLE/UWB to golden belt node only |
| Bracelet | ❌ No | ❌ No | BLE/UWB to golden belt node only |
| Anklet | ❌ No | ❌ No | BLE/UWB to golden belt node only |
| Eyewear | ❌ No | ❌ No | BLE/UWB to golden belt node only |
| **Belt Node** | ✅ Yes | ✅ Optional | Direct WiFi/Cellular testing |

**No non-belt node should ever appear on WiFi or cellular networks.** Tests verify this architectural constraint.

---

## 3. Hardware Architecture

### 3.1 Carrier Board (Universal)

| Component | Specification | Purpose |
|-----------|---------------|---------|
| Test Controller MCU | STM32H7 or RP2040 | Real-time signal generation and capture |
| Host SBC | Raspberry Pi 5 or x86 mini PC | Runs test suite, logs results, serves UI |
| Programmable Power | 0–5V, 0–3A source/measure | Node power profiling per mode |
| Current Sense × 3 | INA219 (main + V_CAM + V_UWB) | Per-rail power measurement |
| Precision Timing | 1 µs resolution timer | Latency measurement for extreme velocity |
| RF Shielding | Aluminum can with RF absorber | Isolates DUT during radar/radio tests |
| Communications | USB, UART, SWD, I2C, SPI | Firmware flashing, debug, API |
| Antenna Diversity Switch | SPDT RF switch + 2 test antennas | Antenna diversity validation |
| UWB Golden Node | Qorvo DW3000 reference design | UWB timing and bandwidth validation |
| BLE Golden Node | nRF52840 or nRF5340 reference | BLE protocol and scheduling validation |

### 3.2 Test Shields Per Node Class

| Shield | Key Equipment |
|--------|---------------|
| Pendant Standard Shield | Radar reflector, LRA reference microphone, camera chart, acoustic speaker |
| Pendant 360° Shield | 8-position LED array, curved calibration chart, stitching verification target, FSYNC timing analyzer |
| Pendant Event-Enhanced Shield | All 360° equipment + event camera strobe array for extreme velocity |
| Bracelet Shield | Radar absorber, LRA resonance fixture, camera chart (camera variants), antenna diversity test fixture |
| Belt Shield | WiFi test AP, cellular signal simulator, thermal load fixture, antenna diversity analyzer, BAN golden node |
| Anklet Shield | Calibrated ToF target (0.3 m), LRA resonance fixture, IMU impact simulator, antenna diversity fixture |
| Eyewear Shield | Event camera LED strobe (< 100 µs), IMU gimbal, forward camera chart, antenna diversity fixture |

### 3.3 Specialized Test Equipment

| Equipment | Purpose | Shield Used |
|-----------|---------|-------------|
| Doppler velocity simulator | High-speed target for mmWave extreme velocity | Pendant, Belt |
| Event camera strobe array | Sub-millisecond illumination for event sensor | Pendant Event-Enhanced, Eyewear |
| Antenna diversity analyzer | Measures RSSI from multiple antenna positions | Belt, Bracelet, Anklet, Eyewear |
| Thermal chamber (mini) | Validates thermal throttling | Belt (Linux SoM variants) |
| IP67 submersion fixture | 1m depth, 30 minutes | All wearable nodes |
| RF spectrum analyzer | Measures RF coexistence and interference | Belt |

---

## 4. Test Sequences — All Nodes

### 4.1 Mandatory Tests (All Nodes)

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 1 | Power-on self-test | MCU boots, CRC verified | Firmware integrity |
| 2 | Rail verification | All rails within ±5% | 3.3V, 1.8V, V_CAM if populated |
| 3 | Sensor enumeration | All I2C/SPI sensors respond | Presence, IMU, environmental |
| 4 | IMU gravity vector | 9.81 ± 0.2 m/s² at rest | Sensor fusion baseline |
| 5 | BLE connectivity | Appears on BAN, responds within 200 ms | Via golden belt node |
| 6 | Battery level | 4.1–4.2 V on full charge | Fuel gauge reading |
| 7 | Firmware version | Matches current published build hash | Reproducible builds |
| 8 | Configuration load | Loads `sentinel-wear.toml` correctly | All parameters parsed |
| 9 | **No WiFi verification** | WiFi interface absent/not responding | Architecture constraint |
| 10 | **No Cellular verification** | Cellular interface absent/not responding | Architecture constraint |

### 4.2 BLE Protocol Tests (All Nodes)

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 11 | Connection interval | Matches configured value (±2 ms) | Scheduling verification |
| 12 | Packet latency | Detection message arrives at golden node < 30 ms | Standard QoS |
| 13 | Critical QoS latency | Critical message arrives < 5 ms | Extreme velocity test mode |
| 14 | Schedule slot adherence | Messages sent only in assigned slots | Determinism verification |
| 15 | Retry behavior | < 5% packet retry rate in standard conditions | Link quality baseline |
| 16 | Connection stability | Maintains connection for 60 seconds without disconnect | Reliability |

### 4.3 UWB Tests (Nodes with UWB Populated)

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 17 | UWB initialization | Module responds to init sequence | |
| 18 | Time synchronization | Clock offset < 100 ns vs golden node | PTP-style exchange |
| 19 | Ranging accuracy | Distance to golden node accurate within ±5 cm | |
| 20 | Bandwidth test | Sustained throughput > 4 Mbps | Test payload stream |
| 21 | Burst bandwidth | Peak throughput > 6 Mbps for 1 second | |
| 22 | UWB role verification | Operates in configured role | `timing_only` / `timing_and_ranging` / `full_bandwidth` |

### 4.4 Antenna Diversity Tests (Nodes with Diversity Populated)

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 23 | Antenna switch functional | MCU can switch between antennas | GPIO control verified |
| 24 | RSSI difference detection | Detects ≥ 5 dB difference between antennas | Sensitivity test |
| 25 | Diversity benefit | In body-shadowing condition, diversity improves RSSI by ≥ 3 dB | Real-world simulation |
| 26 | Switch speed | Antenna switch completes < 1 µs | Performance |
| 27 | Hysteresis | Doesn't switch more than once per 100 ms stable condition | Stability |

---

## 5. Pendant Node Tests

### 5.1 Standard and Medallion Variants

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 28 | Radar detection at 0.5 m | Detection event within 500 ms | |
| 29 | Radar detection at 3.0 m | Detection event within 1 second | Range test |
| 30 | Acoustic DOA | Speaker at 90°, error < ±15° | Beamforming |
| 31 | Acoustic material classification | Glass break vs wood impact classified correctly | Full echo analysis |
| 32 | Microphone array sync | All mics sample synchronously | |
| 33 | Environmental readings | Temperature, humidity within ±5% of reference | |

### 5.2 Camera Tests (If Camera Installed)

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 34 | Camera initialization | Module responds within 2 seconds | |
| 35 | Image capture | Frame captured to buffer | Basic function |
| 36 | Resolution verification | Matches configured resolution | |
| 37 | Frame rate verification | Matches configured frame rate | |

### 5.3 Data Handling Mode Verification (All Camera-Equipped Nodes)

**Each data handling mode tested sequentially:**

| Mode | Test | Criteria |
|------|------|----------|
| Metadata-only | Trigger capture → Check mesh output | Only classification tag transmitted; no SD file created; no stream initiated |
| Local storage | Trigger capture → Check SD card | Recording file created on SD within 500 ms of trigger |
| App streaming | Enable streaming → Check belt receipt | Belt controller receives video stream within 2 seconds of activation |
| Full recording | Trigger capture → Check storage | Recording file on SD with correct duration; integrity manifest generated |

**Configuration for testing:**
```toml
# Applied sequentially by test harness
[data]
store_raw_video = [false, true, true, true]
storage_target = [null, "sd_card", "belt_stream", "sd_card"]
recording_trigger = "test_mode"
```

### 5.4 Hardware Switch Tests (Only If Switch Physically Installed)

**Precondition check:** Test reads `CAM_HW_SW_TEST` GPIO. If GPIO not populated, skip these tests.

| # | Test | Criteria |
|---|------|----------|
| 38 | Switch DISABLED | V_CAM rail = 0 V ± 0.05 V. **FAIL if > 0.1 V.** |
| 39 | Switch DISABLED | Camera not initialized (firmware reports error or skips init) |
| 40 | Switch ENABLED | V_CAM rail > 3.0 V |
| 41 | Switch ENABLED | Camera initializes successfully |
| 42 | Switch toggle speed | State change detected within 100 ms of physical toggle |

**Note:** Units without hardware switch proceed directly to software-only privacy control tests. Hardware switch is optional, not mandatory.

### 5.5 Software Privacy Control Tests (All Camera-Equipped Nodes)

| # | Test | Criteria |
|---|------|----------|
| 43 | Software disable | Camera disabled via config → No frames captured |
| 44 | Schedule-based disable | Camera disabled during configured hours → No frames captured |
| 45 | Detection-triggered enable | Camera disabled by default, enabled on detection trigger |
| 46 | Always-on enable | Camera continuously active → Frames captured continuously |

---

## 6. Pendant 360° Curved Variant Tests

### 6.1 Camera Array Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 47 | All cameras initialize | All N cameras (4/6/8) respond within 5 seconds | |
| 48 | Camera frame sync | FSYNC pulse triggers all cameras within ±1 ms | Hardware sync critical |
| 49 | Individual camera frames | Each camera produces frames at configured rate | |
| 50 | MIPI lane integrity | No bit errors on any MIPI lane | Signal integrity |
| 51 | Camera calibration load | Calibration JSON loaded from flash | |

### 6.2 Stitching Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 52 | Stitching initiation | Vision processor begins stitching within 2 seconds of camera init | |
| 53 | Stitch seam quality | LED array at 8 positions → Straight reference lines continuous across boundaries | Seam error < 3 pixels |
| 54 | Equirectangular output | Correct aspect ratio (2:1 width:height) | Panorama format |
| 55 | Frame rate | Stitched output matches configured rate | |
| 56 | Latency (on-pendant stitching) | Stitched frame available within 100 ms of capture | |
| 57 | Latency (belt stitching) | Raw frames arrive at belt within 50 ms | Belt stitches |

### 6.3 Tiered Progressive Quality Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 58 | QVGA baseline available | All 8 cameras produce QVGA stream within 1 second | Instant baseline |
| 59 | Progressive refinement | Higher-res keyframes sent sequentially | Quality improves over time |
| 60 | Quality plateau | Quality stabilizes within 10 seconds | |
| 61 | Bandwidth measurement | Total bandwidth within configured UWB limit | Max 5-6 Mbps for UWB |

### 6.4 360° Recording Tests

| # | Test | Criteria |
|---|------|----------|
| 62 | Recording initiation | 360° recording starts on trigger |
| 63 | Recording duration | 30-second test → file duration 30.0 ± 0.5 seconds |
| 64 | File size | > 0 bytes, reasonable for duration and quality |
| 65 | SHA-256 integrity | Hash computed and stored in manifest |
| 66 | Playback verification | Recording plays back without artifacts |

### 6.5 Companion App Streaming Tests (Via Belt Node)

| # | Test | Criteria |
|---|------|----------|
| 67 | Stream initiation | 360° stream available in companion app within 5 seconds of request |
| 68 | Stream quality | Matches configured quality level |
| 69 | Stream stability | Stream continues for 60 seconds without interruption |
| 70 | Multi-client test | 2 simultaneous clients receive stream |

---

## 7. Bracelet Node Tests

### 7.1 Sensing Tests

| # | Test | Criteria |
|---|------|----------|
| 71 | Radar detection at 0.3 m | Detection event within 300 ms |
| 72 | Radar detection at 1.0 m | Detection event within 500 ms |
| 73 | IMU orientation | Orientation matches expected angle at rest |
| 74 | IMU motion detection | Detects rapid motion (shake test) |

### 7.2 Haptic Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 75 | LRA resonance | 150 ± 15 Hz resonance detected | Acoustic measurement |
| 76 | Haptic current draw | 300–500 mA peak at full amplitude | |
| 77 | Pattern playback | Test pattern plays without error | DRV2605L waveform |
| 78 | Pattern timing | Pattern duration matches specification | |
| 79 | Direction encoding | Correct haptic pattern for each alert class | Alert routing |

### 7.3 Camera Tests (Variant C)

Same as Section 5.2-5.5 for camera-equipped nodes.

### 7.4 UWB Tests (If Populated)

Same as Section 4.3 for UWB-equipped nodes.

---

## 8. Belt Node Tests

### 8.1 Unique Architectural Role Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 80 | **WiFi presence verified** | WiFi interface present and functional | Belt is ONLY node with WiFi |
| 81 | **No WiFi on other nodes** | Verify non-belt nodes have no WiFi connectivity | Architecture constraint |
| 82 | BAN hub functionality | Receives from all connected nodes | Coordinator role |
| 83 | Time master | Serves time sync to all nodes | Clock authority |

### 8.2 WiFi Tests

| # | Test | Criteria |
|---|------|----------|
| 84 | AP association | Connects to test AP within 30 seconds |
| 85 | RSSI measurement | RSSI > -70 dBm at 1 m from test AP |
| 86 | API server startup | Embedded HTTP server responds within 5 seconds |
| 87 | REST API endpoints | All basic endpoints return 200 OK |
| 88 | WebSocket connection | WebSocket `/ws/events` connects within 2 seconds |
| 89 | Media streaming | RTSP stream opens within 5 seconds |
| 90 | mDNS discovery | `sentinel-wear.local` resolves correctly |
| 91 | WiFi bandwidth | Throughput > 50 Mbps to test client |
| 92 | 5 GHz band preference | Associates on 5 GHz when available |

### 8.3 Cellular Tests (If Module Installed)

| # | Test | Criteria |
|---|------|----------|
| 93 | SIM detection | `SIM_DET` GPIO reads card present |
| 94 | SIM power rail | `SIM_VCC` within spec (1.8V or 3.0V per module) |
| 95 | Module power-on | Cellular module powers on within 5 seconds |
| 96 | AT command response | `AT` returns `OK` within 5 seconds |
| 97 | Network registration | Module reports registered within 60 seconds (requires active SIM) |
| 98 | Signal strength | RSSI reported via AT command |
| 99 | Carrier identification | Carrier name reported correctly |
| 100 | Data connection | PDP context established successfully |
| 101 | Throughput test | Cellular bandwidth within spec for module class |
| 102 | Alert delivery | Test alert delivered to configured endpoint via cellular |

**Note on SIM testing:** Full network registration tests require a live SIM with active data plan. Module-level tests (AT commands) can be performed with inactive SIM.

### 8.4 BLE Direct to Companion App

| # | Test | Criteria |
|---|------|----------|
| 103 | BLE advertising | Belt advertises as `sentinel-wear` |
| 104 | BLE connection | Companion app test client connects via BLE |
| 105 | BLE API latency | REST request via BLE responds within 200 ms |
| 106 | BLE bandwidth | Metadata and alerts transmitted successfully |

### 8.5 Fusion and SLAM Tests

| # | Test | Criteria |
|---|------|----------|
| 107 | PentaTrack benchmark | 1000-frame update < 10 ms/frame |
| 108 | SLAM initialization | SLAM system initializes within 10 seconds |
| 109 | SLAM map building | Test walk-through generates map with < 5 cm drift |
| 110 | Loop closure | Revisiting location triggers loop closure correctly |

### 8.6 Recording Manager Tests

| # | Test | Criteria |
|---|------|----------|
| 111 | Recording index | `RecordingAvailable` event indexed correctly |
| 112 | Recording retrieval | Recording accessible via API |
| 113 | Integrity manifest | SHA-256 hash matches file content |
| 114 | Legal export | Export ZIP contains file + manifest |
| 115 | Retention enforcement | Old recordings deleted after retention period |

### 8.7 Thermal Tests

| # | Test | Criteria |
|---|------|----------|
| 116 | Idle temperature | Enclosure temperature < 35°C at idle |
| 117 | Load temperature | Enclosure temperature < 45°C at full compute load |
| 118 | Thermal throttling | SLAM frame rate reduces when > 45°C |
| 119 | Thermal shutdown | System shuts down if > 50°C (safety) |

### 8.8 Power Budget Tests

| # | Test | Criteria |
|---|------|----------|
| 120 | Sleep current | Within spec for configured sleep mode |
| 121 | Idle current | Within spec for idle operation |
| 122 | Active current | Within spec for active operation |
| 123 | Peak current | Within spec for peak transient |
| 124 | Battery hot-swap (Variant E) | Continues operation during battery swap |

### 8.9 Antenna Diversity Tests (If Populated)

| # | Test | Criteria |
|---|------|----------|
| 125 | BLE antenna switch | Both BLE antennas functional |
| 126 | WiFi antenna switch | Both WiFi antennas functional (if dual WiFi) |
| 127 | Diversity RSSI improvement | Diversity mode improves RSSI by ≥ 3 dB in body-shadow condition |
| 128 | Seamless handoff | Switching between antennas doesn't drop connection |

---

## 9. Anklet Node Tests

### 9.1 ToF/LiDAR Tests

| # | Test | Criteria |
|---|------|----------|
| 129 | ToF at 0.3 m | Reading within ±5 cm |
| 130 | ToF at 1.0 m | Reading within ±10 cm |
| 131 | ToF at 2.0 m | Reading within ±15 cm |

### 9.2 Gait Analysis Tests

| # | Test | Criteria |
|---|------|----------|
| 132 | IMU heel-strike | ≥ 3 g recorded on simulated heel-strike |
| 133 | Step detection | 10 simulated steps detected correctly |
| 134 | Cadence measurement | Within ±5% of reference |
| 135 | Stride asymmetry | Left/right difference detected if present |
| 136 | Gait phase identification | Stance/swing/strike phases identified |

### 9.3 Haptic Tests

Same as Section 7.2 for haptic testing.

### 9.4 UWB Gait Sync Tests (If UWB Populated)

| # | Test | Criteria |
|---|------|----------|
| 137 | Left-right time sync | Anklet pair synchronized within 1 ms |
| 138 | Gait phase sync | Gait phases aligned between anklets |

---

## 10. Eyewear Node Tests

### 10.1 Event Camera Tests

| # | Test | Criteria |
|---|------|----------|
| 139 | Event response to strobe | Event burst received within 1 ms of LED flash |
| 140 | Event latency | < 100 µs to first event |
| 141 | Event rate | Handles > 10M events/second without overflow |

### 10.2 IMU Tests

| # | Test | Criteria |
|---|------|----------|
| 142 | IMU 200 Hz operation | No dropped samples over 10 seconds |
| 143 | Head orientation tracking | Tracks head rotation independently from torso |

### 10.3 Head-Torso Separation Tests

| # | Test | Criteria |
|---|------|----------|
| 144 | Head frame independence | Eyewear IMU reported independently from belt IMU |
| 145 | Coordinate transform | Detection from eyewear correctly transformed to body frame |

---

## 11. Extreme Velocity Detection Tests

### 11.1 Doppler Radar Tests

| # | Test | Criteria | Notes |
|---|------|----------|-------|
| 146 | Extended Doppler configuration | Radar configured for > 300 m/s velocity range | Custom chirp profile |
| 147 | High update rate | > 10 kHz Doppler updates | Continuous wave mode |
| 148 | Fast target detection | Detects simulated 100 m/s target | Velocity calibration |
| 149 | Detection latency | Detection reported < 100 µs from RF event | |

### 11.2 Event Camera Integration Tests

| # | Test | Criteria |
|---|------|----------|
| 150 | Event trigger on Doppler | Event camera triggers when Doppler threshold exceeded |
| 151 | Combined latency | Total detection latency < 1 ms |
| 152 | Trajectory capture | Event camera captures trajectory streak |

### 11.3 Critical QoS Tests

| # | Test | Criteria |
|---|------|----------|
| 153 | Critical message preemption | Extreme velocity alert preempts other BLE traffic |
| 154 | Alert latency end-to-end | Alert reaches companion app < 10 ms from detection |
| 155 | Haptic alert timing | Haptic fires within 5 ms of alert received |

### 11.4 UWB Burst Test

| # | Test | Criteria |
|---|------|----------|
| 156 | Event camera data burst | Event camera data sent via UWB burst |
| 157 | Burst bandwidth | Burst throughput > 5 Mbps |
| 158 | Burst completion | Burst completes within 100 ms |

---

## 12. IP67 Test Sequence

### 12.1 Pre-Submersion

| # | Test | Record |
|---|------|--------|
| 159 | Weight | Record dry weight |
| 160 | Rail voltages | Record baseline voltages |
| 161 | Sensor function | Record baseline readings |

### 12.2 Submersion

| Parameter | Value |
|-----------|-------|
| Depth | 1.0 m |
| Duration | 30 minutes |
| Temperature | 25°C ± 2°C |

### 12.3 Post-Submersion

| # | Test | Criteria |
|---|------|----------|
| 162 | Power-on | Node powers on within 60 seconds |
| 163 | Sensor function | All sensors initialize |
| 164 | BAN connectivity | Appears on network |
| 165 | Visual inspection | No moisture visible inside test window |
| 166 | Rail voltages | Within ±2% of pre-submersion values |
| 167 | Weight | Within 0.1 g of pre-submersion (no water ingress) |

---

## 13. RF Coexistence Tests

### 13.1 BLE + WiFi Coexistence

| # | Test | Criteria |
|---|------|----------|
| 168 | BLE during WiFi idle | BLE latency unchanged |
| 169 | BLE during WiFi streaming | BLE latency increase < 20% |
| 170 | WiFi band preference | System prefers 5 GHz over 2.4 GHz |

### 13.2 UWB Coexistence

| # | Test | Criteria |
|---|------|----------|
| 171 | UWB + BLE simultaneous | Both operate without interference |
| 172 | UWB + WiFi simultaneous | Both operate without interference |

### 13.3 Antenna Isolation

| # | Test | Criteria |
|---|------|----------|
| 173 | BLE to WiFi isolation | > 20 dB isolation between antennas |
| 174 | BLE to Cellular isolation | > 20 dB isolation between antennas |

---

## 14. Thermal Management Tests

### 14.1 Steady-State Thermal

| # | Test | Criteria |
|---|------|----------|
| 175 | Idle temperature | Enclosure < 35°C |
| 176 | Active temperature | Enclosure < 45°C during sustained operation |

### 14.2 Thermal Throttling

| # | Test | Criteria |
|---|------|----------|
| 177 | Throttle activation | SLAM rate reduces at 45°C |
| 178 | Throttle effectiveness | Temperature stabilizes after throttle |
| 179 | Temperature recovery | Temperature decreases after load removed |

### 14.3 360° Pendant Thermal

| # | Test | Criteria |
|---|------|----------|
| 180 | Pendant surface temp | < 45°C during full 360° operation |
| 181 | Skin-contact temp | < 42°C |
| 182 | Vision processor temp | < 85°C junction |

---

## 15. Software Test Runner

### 15.1 Test Harness Architecture

```
sentinel-test-harness/
├── src/
│   ├── main.rs
│   ├── carrier_board.rs      # Hardware abstraction
│   ├── test_shield.rs        # Shield-specific tests
│   ├── tests/
│   │   ├── electrical.rs
│   │   ├── sensing.rs
│   │   ├── connectivity.rs
│   │   ├── data_handling.rs
│   │   ├── privacy.rs
│   │   ├── extreme_velocity.rs
│   │   └── environmental.rs
│   └── report.rs
└── Cargo.toml
```

### 15.2 Test Execution Flow

```
1. Carrier board self-test
2. Shield detection (identify which shield is installed)
3. DUT connection (via test pads or wireless)
4. Firmware version verification
5. Electrical tests
6. Sensing tests
7. Connectivity tests
8. Data handling tests
9. Privacy control tests
10. Performance tests
11. Report generation
```

### 15.3 JSON Output Format

```json
{
  "test_run_id": "TR-20240315-143022",
  "test_station_id": "UTS-001",
  "operator": "automated",
  "dut": {
    "serial": "SW-PND360-00012",
    "node_type": "Pendant360",
    "hardware_variant": "8-camera",
    "firmware_rev": "v1.0.0",
    "build_hash": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2"
  },
  "test_configuration": {
    "data_handling_mode": "local_storage",
    "uwb_enabled": true,
    "camera_count": 8,
    "hardware_switch_installed": false
  },
  "tests": [
    {"name": "Power_Rails", "status": "PASS", "value": "3.31V", "duration_ms": 12},
    {"name": "BLE_Connectivity", "status": "PASS", "rssi_dbm": -41, "latency_ms": 18},
    {"name": "Camera_Array_Init", "status": "PASS", "cameras_found": 8, "duration_ms": 4234},
    {"name": "FSYNC_Sync_Accuracy", "status": "PASS", "max_delta_ms": 0.4},
    {"name": "Stitch_Quality", "status": "PASS", "max_seam_error_px": 2.1},
    {"name": "UWB_Time_Sync", "status": "PASS", "offset_ns": 67},
    {"name": "Recording_360", "status": "PASS", "file_size_bytes": 45231088, "sha256": "a3f4..."},
    {"name": "Critical_QoS_Latency", "status": "PASS", "latency_ms": 4.2},
    {"name": "Extreme_Velocity_Detection", "status": "PASS", "detection_latency_us": 89},
    {"name": "IP67_PostSubmersion", "status": "PASS", "rail_deviation_pct": 0.3}
  ],
  "measurements": {
    "power_draw_active_mw": 3200,
    "thermal_enclosure_temp_c": 38.2,
    "ble_latency_avg_ms": 22.3,
    "uwb_bandwidth_mbps": 4.8
  },
  "final_result": "PASS",
  "duration_seconds": 142
}
```

---

## 16. Test Report Format (Human-Readable)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    SENTINEL-WEAR TEST REPORT                                  ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  Serial:        SW-PND360-00012                                               ║
║  Node Type:     Pendant360                                                    ║
║  Variant:       8-camera                                                      ║
║  Firmware:      v1.0.0                                                        ║
║  Test Date:     2024-03-15 14:30:22 UTC                                       ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  ELECTRICAL TESTS                                                             ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] Power Rails ................... 3.31V                                 ║
║  [PASS] Battery Level ................. 4.15V                                 ║
║  [PASS] Current Draw .................. 320 mW active                         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  CONNECTIVITY TESTS                                                           ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] BLE Connectivity .............. RSSI -41 dBm, Latency 18 ms           ║
║  [PASS] UWB Time Sync ................. Offset 67 ns                          ║
║  [PASS] UWB Bandwidth ................. 4.8 Mbps sustained                    ║
║  [PASS] Antenna Diversity ............. Both antennas functional              ║
║  [SKIP] WiFi Test ..................... Not applicable (non-belt node)        ║
║  [SKIP] Cellular Test ................. Not applicable (non-belt node)         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  SENSOR TESTS                                                                 ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] IMU Gravity Vector ............. 9.79 m/s²                            ║
║  [PASS] Radar Detection ............... Target detected at 0.5 m              ║
║  [PASS] Acoustic DOA .................. Error 12°                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  360° CAMERA TESTS                                                            ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] Camera Array Init .............. 8/8 cameras initialized              ║
║  [PASS] FSYNC Sync Accuracy ........... Max delta 0.4 ms                      ║
║  [PASS] Stitch Quality ................ Max seam error 2.1 px                 ║
║  [PASS] 360° Recording ................ 45.2 MB, SHA-256 verified             ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  EXTREME VELOCITY TESTS                                                       ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] Doppler Extended Range ......... Configured for > 300 m/s             ║
║  [PASS] Fast Target Detection ......... 100 m/s target detected              ║
║  [PASS] Detection Latency .............. 89 µs                                 ║
║  [PASS] Critical QoS Latency .......... 4.2 ms to belt                         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  DATA HANDLING TESTS                                                          ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] Metadata-Only Mode ............. Only tags transmitted                ║
║  [PASS] Local Storage Mode ............. SD file created                      ║
║  [PASS] App Streaming Mode ............. Belt received stream                 ║
║  [PASS] Integrity Chain ................ SHA-256 verified                      ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  PRIVACY CONTROL TESTS                                                        ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [SKIP] Hardware Switch ................. Not installed                        ║
║  [PASS] Software Disable ............... Camera disabled via config           ║
║  [PASS] Schedule-Based Disable ......... Verified                             ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  ENVIRONMENTAL TESTS                                                          ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  [PASS] Thermal ........................ Enclosure 38.2°C                     ║
║  [PASS] IP67 Post-Submersion .......... Rail deviation 0.3%                    ║
╠══════════════════════════════════════════════════════════════════════════════╣
║  SUMMARY                                                                      ║
║  ──────────────────────────────────────────────────────────────────────────────║
║  Tests Run:     32                                                            ║
║  Tests Passed:  30                                                            ║
║  Tests Skipped: 2                                                             ║
║  Tests Failed:  0                                                             ║
║  Duration:      142 seconds                                                   ║
║                                                                                ║
║  ████████████████████████████████████████████ RESULT: PASS                     ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 17. Belt Node Complete Test Example

```json
{
  "test_run_id": "TR-20240315-150822",
  "test_station_id": "UTS-001",
  "operator": "automated",
  "dut": {
    "serial": "SW-BLT-00018",
    "node_type": "Belt",
    "hardware_variant": "Linux_SoM",
    "firmware_rev": "v1.0.0",
    "build_hash": "b7e8d1f3a2c4e5f6"
  },
  "test_configuration": {
    "wifi_band": "5GHz",
    "cellular_enabled": true,
    "cellular_module": "Quectel EC25",
    "uwb_enabled": true,
    "antenna_diversity": true
  },
  "tests": [
    {"name": "Power_Rails", "status": "PASS", "value": "3.30V"},
    {"name": "WiFi_AP_Connect", "status": "PASS", "ssid": "TestAP", "band": "5GHz", "rssi_dbm": -35},
    {"name": "WiFi_Band_Preference", "status": "PASS", "preferred_band": "5GHz"},
    {"name": "API_Server", "status": "PASS", "endpoints_tested": 15},
    {"name": "WebSocket_Stream", "status": "PASS", "latency_ms": 12},
    {"name": "Cellular_SIM_Detected", "status": "PASS"},
    {"name": "Cellular_AT_Response", "status": "PASS", "module": "Quectel EC25"},
    {"name": "Cellular_Network_Registration", "status": "PASS", "carrier": "TestCarrier"},
    {"name": "Cellular_Throughput", "status": "PASS", "dl_mbps": 45.2, "ul_mbps": 12.3},
    {"name": "BLE_Coordinator", "status": "PASS", "nodes_connected": 4},
    {"name": "BLE_Direct_App", "status": "PASS", "latency_ms": 45},
    {"name": "UWB_Time_Sync", "status": "PASS", "offset_ns": 52},
    {"name": "Antenna_Diversity_BLE", "status": "PASS", "rssi_improvement_db": 8.2},
    {"name": "Antenna_Diversity_WiFi", "status": "PASS", "rssi_improvement_db": 6.5},
    {"name": "Fusion_Benchmark", "status": "PASS", "ms_per_frame": 6.2},
    {"name": "SLAM_Init", "status": "PASS", "init_time_s": 3.2},
    {"name": "Thermal_Idle", "status": "PASS", "temp_c": 32.1},
    {"name": "Thermal_Load", "status": "PASS", "temp_c": 41.8},
    {"name": "Thermal_Throttle", "status": "PASS", "throttle_triggered": false},
    {"name": "Recording_Manager", "status": "PASS", "recordings_indexed": 3},
    {"name": "Integrity_Export", "status": "PASS", "manifest_verified": true},
    {"name": "RF_Coexistence_BLE_WiFi", "status": "PASS", "latency_increase_pct": 8.3},
    {"name": "No_WiFi_Other_Nodes", "status": "PASS", "wifi_detected": false}
  ],
  "measurements": {
    "power_draw_idle_mw": 850,
    "power_draw_active_mw": 4200,
    "power_draw_streaming_mw": 5100,
    "thermal_enclosure_temp_c": 41.8,
    "ble_latency_avg_ms": 18.5,
    "wifi_throughput_mbps": 142.3,
    "cellular_latency_avg_ms": 45
  },
  "final_result": "PASS",
  "duration_seconds": 245
}
```

---

## 18. Safety and Compliance

### 18.1 Battery Safety

- **Charging profiles enforced by test jig**
- **Temperature monitoring during charge cycles**
- **Overcharge/overdischarge protection verified**
- **Do not leave batteries unattended during extended tests**

### 18.2 ESD Precautions

- **ESD-safe work surface required**
- **Grounded wrist strap when handling bare PCBs**
- **ESD-safe packaging for DUTs**

### 18.3 RF Safety

- **Test performed in RF shielded environment**
- **Power levels within FCC Part 15 limits**
- **Maintain safe distance from radiating antennas**

### 18.4 Thermal Safety

- **Do not touch enclosure during thermal tests**
- **Allow cooling period after high-load tests**
- **Thermal shutdown verified functional**

---

## 19. Test Skip Conditions

The following tests are skipped based on DUT configuration:

| Test | Skip Condition | Rationale |
|------|----------------|-----------|
| Camera tests | Camera not installed | No camera hardware present |
| Hardware switch tests | Switch not installed | Optional feature |
| UWB tests | UWB not populated | Optional feature |
| WiFi tests (non-belt) | Non-belt node | Architecture: belt-only WiFi |
| Cellular tests (non-belt) | Non-belt node | Architecture: belt-only cellular |
| Antenna diversity tests | Single antenna | Diversity not populated |
| Extreme velocity tests | Feature disabled | Opt-in feature |

---

## 20. Test Duration Estimates

| Node Type | Variant | Full Test Suite Duration |
|-----------|---------|-------------------------|
| Pendant Standard | All | 3-5 minutes |
| Pendant 360° | 8-camera | 5-8 minutes |
| Pendant Event-Enhanced | All | 6-9 minutes |
| Bracelet | All | 2-4 minutes |
| Belt | MCU | 5-8 minutes |
| Belt | Linux SoM | 8-12 minutes |
| Belt | With Cellular | 10-15 minutes |
| Anklet | All | 2-4 minutes |
| Eyewear | All | 3-5 minutes |
| IP67 additional | Any | +5 minutes |

---

## 21. Calibration Requirements

### 21.1 Test Equipment Calibration Schedule

| Equipment | Calibration Interval |
|-----------|---------------------|
| Programmable power supply | 12 months |
| Current sense modules | 12 months |
| Oscilloscope | 12 months |
| Spectrum analyzer | 12 months |
| Reference sensors (temperature, humidity) | 6 months |
| Calibrated targets (ToF, camera charts) | 24 months or after visible wear |

### 21.2 Golden Node Requirements

| Golden Node | Purpose | Verification |
|-------------|---------|---------------|
| BLE Golden Node | Reference coordinator | Verified against reference implementation |
| UWB Golden Node | Time sync reference | Calibrated timing reference |
| WiFi Test AP | Network simulation | Standard WiFi AP with known performance |

---

## 22. Production Integration

### 22.1 Test Station Setup

```
Test Station UTS-001:
├── Carrier Board (STM32H7 + RP5)
├── Test Shields (one per node type in rotation)
├── RF Shield Enclosure
├── IP67 Submersion Fixture
├── Thermal Chamber (mini)
├── Reference Calibration Targets
├── Golden Nodes (BLE + UWB)
└── Test Software (sentinel-test-harness)
```

### 22.2 Test Queue Management

```
1. Operator scans node serial barcode
2. System identifies node type from database
3. Shield recommendation displayed
4. Operator installs correct shield
5. Automated test sequence runs
6. Report generated and stored
7. Pass/Fail label printed
8. Node routed to next station or rework
```

### 22.3 Statistical Process Control

- Pass rate tracked per node type
- Failure modes logged and categorized
- Trend analysis for yield improvement
- Per-station calibration drift monitoring

---

## 23. Directory Structure

```
hardware/testing/test_jig_pcb/
├── carrier_board/
│   ├── carrier_board.kicad_pro
│   ├── carrier_board.kicad_sch
│   ├── carrier_board.kicad_pcb
│   └── gerbers/
├── test_shields/
│   ├── pendant_standard_shield/
│   ├── pendant_360_shield/
│   ├── pendant_event_enhanced_shield/
│   ├── bracelet_shield/
│   ├── belt_shield/
│   ├── anklet_shield/
│   └── eyewear_shield/
├── fixtures/
│   ├── ip67_submersion/
│   ├── thermal_chamber/
│   └── rf_shield_enclosure/
├── golden_nodes/
│   ├── ble_golden_node/
│   └── uwb_golden_node/
├── calibration_targets/
│   ├── tof_target_0.3m/
│   ├── camera_chart_standard/
│   ├── camera_chart_360/
│   └── stitching_verification/
├── test_equipment_bom/
│   └── test_equipment_list.md
├── test_software/
│   └── sentinel-test-harness/    # Rust test runner
├── production_sop/
│   ├── test_station_setup.md
│   ├── daily_calibration_check.md
│   └── test_procedure.md
└── README.md
```

---

**End of Test Jig PCB Reference Document**
