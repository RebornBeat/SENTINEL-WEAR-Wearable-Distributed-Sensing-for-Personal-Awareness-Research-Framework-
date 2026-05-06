# Belt Node — Primary Compute, Hub, and App Server

**Project:** SENTINEL-WEAR
**Node Position:** Waist belt / belt clip
**Primary Role:** Compute hub, battery hub, torso IMU reference, BAN coordinator, app server, recording manager
**Status:** Reference Design — All Variants

---

## 1. Role in the Mesh

The Belt Node is the central coordination unit of the SENTINEL-WEAR system.

**Primary Functions:**
- **Compute Hub:** Runs `sentinel-fusion`, `sentinel-tracking` (PentaTrack bridge), `sentinel-slam` (Linux SoM variant), and all system coordination.
- **Torso Reference:** The belt IMU defines the body-frame origin (+Y forward, +X right, +Z up). All other node orientations are expressed relative to this.
- **BAN Hub:** Routes all inter-node BAN traffic. Receives detections, gait events, acoustic events, identification results, and recording notifications from all nodes.
- **App Server:** Runs the embedded HTTP/WebSocket server for the SENTINEL-WEAR companion app. Serves the REST API, WebSocket event stream, and media streams over Wi-Fi or BLE. See `apps/README.md`.
- **Recording Manager:** Manages recording lifecycle — capture, storage to SD card or internal flash, retention enforcement, and legal export. Receives `RecordingAvailable` events from all nodes; aggregates and indexes all recordings for companion app access.
- **Sensing:** mmWave radar (downward, torso-level) for independent ground-plane and proximity coverage.
- **SLAM Processor (Linux SoM variant):** Runs 360° panorama stitching from curved pendant cameras; runs SLAM pipeline from LiDAR and camera data for dense world model.

---

## 2. Compute Variants — All Supported

### Variant A — MCU (Research Minimum)

**MCU:** STM32H7 class (Cortex-M7, 480 MHz, FPU)

- Runs `sentinel-belt-controller` Rust binary on embedded RTOS.
- Sufficient for: full body-frame fusion (6 nodes), PentaTrack, alert routing, BAN hub, basic recording management.
- Not sufficient for: 360° stitching, dense SLAM, Linux app server.
- Companion app: limited to BLE local connection; no Wi-Fi server.
- ~2000 mAh battery.

**Use case:** Minimal viable system for research, low-power deployment.

### Variant B — Linux SoM (Full Capability)

**SoM:** Raspberry Pi Compute Module 4 (quad-core Cortex-A72, 4–8 GB RAM, eMMC, Wi-Fi, BT)

- Full Linux stack. Runs `sentinel-belt-controller` as a Rust native binary.
- Runs all crates including `sentinel-slam` and `sentinel-api` (axum HTTP server).
- Full companion app server: REST API, WebSocket, RTSP/H.264/360° media streams.
- Handles 360° panorama stitching from curved pendant cameras.
- SLAM-ready: runs ORB-SLAM3 or LIO-SAM with sufficient compute.
- 5000–7000 mAh battery bank.
- Thermal management required (see Section 6).

**Use case:** Full capability: SLAM, 360° stitching, recording management, companion app.

### Variant C — High-Performance SoM

**SoM:** NXP i.MX 8M Plus (quad-core Cortex-A53 + Cortex-M7, NPU 2.3 TOPS)

- All Variant B capabilities.
- NPU enables on-device neural classification for all nodes (centralizes inference, reduces node compute).
- Better efficiency than Variant B for sustained workloads.
- Built-in M7 co-processor handles real-time BAN protocol without interfering with A53 fusion workload.
- 5000–7000 mAh battery bank.

**Use case:** High-performance research deployment. GPU-like NPU for dense model inference.

### Variant D — Minimal (ESP32)

**MCU:** ESP32-S3

- Lowest cost, highest power efficiency.
- Integrated Wi-Fi and BLE.
- Can serve a minimal HTTP API to companion app.
- Limited to 2–3 node fusion (RAM constrained).
- ~2000 mAh.

**Use case:** Low-complexity single-node or dual-node deployments. Cost-optimized research.

---

## 3. MCU Co-Processor (Sensor Interface)

For Variants B and C (Linux SoM), a co-processor handles real-time tasks that require deterministic scheduling:

- **Variant A:** STM32U5 (sensor I/O, BAN radio coordination)
- **Variant B:** nRF5340 network core (BAN protocol handler, BLE time-sync master)

The co-processor pattern separates real-time interrupt handling from OS-level fusion processing.

---

## 4. Downward-Facing mmWave Radar

- **Variant A:** TI IWR6843AOP (60 GHz, AOA, oriented downward from belt buckle)
- **Variant B:** Acconeer XR112 (ultra-compact, lower power)

**Coverage:** Detects objects at ground level approaching the wearer's feet, ground surface for gait analysis cross-reference (combined with anklet IMUs), potential trip hazards.

---

## 5. IMU (Torso Reference — Critical)

- **Variant A:** Bosch BMI270 (6-axis, 0.1 mg noise density, consistent with all other nodes)
- **Variant B:** TDK ICM-42688-P (0.07 mg noise density, smallest package)

The belt IMU noise density and bias stability are the most critical parameters in the system. All other node orientations are expressed relative to this one. Higher quality here benefits the entire body-frame model.

---

## 6. External Interfaces

| Interface | Purpose | Variants |
|---|---|---|
| Wi-Fi 6 (via SoM) | Companion app over home network | B, C |
| Bluetooth 5.3 | Mobile companion app (no Wi-Fi) | All |
| USB-C | PC companion app + firmware update + charging | All |
| LTE (optional module) | Emergency contact when off-network | B, C |
| BAN Radio (nRF52840 / nRF5340) | Body-area network hub — all nodes | All |
| SD card | Recording storage (high-capacity) | All (B/C preferred) |

---

## 7. Thermal Management

For Variants B and C running sustained compute (SLAM, stitching, app server):

- **Thermal dissipation:** Design enclosure with venting or heatspreader for the SoM.
- **Thermal pad:** Required under SoM PCB module footprint.
- **Temperature monitoring:** NTC thermistor on enclosure interior wall; firmware limits compute if temperature exceeds 50°C.
- **Duty cycling:** SLAM pipeline runs at reduced frame rate when thermal budget is tight.

---

## 8. Battery Architecture

- **Variant A, D:** Li-Ion 18650 × 1 (3500 mAh) — ~8–16 hours
- **Variant B, C:** Li-Ion 18650 × 2 parallel (7000 mAh) — ~12–24 hours

**Belt node can optionally power other nodes:**
- Conductive traces through the belt (for nodes with belt-connection power rail).
- This is an option for the curved 360° pendant (which has high power draw).

---

## 9. Form Factor

- **Enclosure:** Belt buckle module or rear clip pack (both supported).
- **Weight:** 100–300 g including battery bank. Worn at hip — weight supported by belt, not the wrist or neck.
- **Attachment:** Removable belt clip or integrated into custom belt.
- **Charging:** USB-C PD preferred for faster charging of high-capacity variants.

---

## 10. No Identification Layer on This Node

The Belt Node does not carry an identification sensor. Identification is the role of the Pendant Node (user-configured). No camera hardware, no optional hardware switch, and no relevant firmware for identification are present on the belt.

---

## 11. Directory Structure

```
belt_node/
├── variant_a_mcu/
│   ├── belt_node_mcu.kicad_pro
│   └── gerbers/
├── variant_b_linux_som/
│   ├── belt_node_cm4.kicad_pro
│   └── gerbers/
├── variant_c_imx8/
│   └── gerbers/
├── variant_d_esp32/
│   └── gerbers/
├── bom/
└── README.md
```
