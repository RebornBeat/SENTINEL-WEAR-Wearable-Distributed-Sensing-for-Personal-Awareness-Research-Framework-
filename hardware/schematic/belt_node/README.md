# Belt Node — Primary Compute, Hub, and App Server

**Project:** SENTINEL-WEAR
**Node Position:** Waist belt / belt clip
**Primary Role:** Compute hub, battery hub, torso IMU reference, BAN coordinator, WiFi/cellular gateway, app server, recording manager
**Status:** Reference Design — All Variants
**License:** CERN-OHL-S v2

---

## 1. Role in the Mesh — The Sole External Gateway

The Belt Node is the central coordination unit of the SENTINEL-WEAR system and the **only node with external network connectivity**. All other nodes (pendant, bracelets, anklets, eyewear) communicate exclusively via BAN (BLE 5.x or UWB) to the belt node. No other node has WiFi or cellular capability.

**Data flow:**
```
Pendant / Bracelets / Anklets / Eyewear
       │
       │  BAN (BLE 5.x or UWB)
       ▼
  Belt Node ──► WiFi (local network) ──► Companion App
               │
               └──► Cellular/SIM (LTE/5G) ──► Companion App (remote)
                                          └──► Emergency Contact
```

**Primary Functions:**
- **Compute Hub:** Runs `sentinel-fusion`, `sentinel-tracking` (PentaTrack bridge), `sentinel-slam` (Linux SoM variant), and all system coordination.
- **Torso Reference:** The belt IMU defines the body-frame origin (+Y forward, +X right, +Z up). All other node orientations are expressed relative to this.
- **BAN Hub:** Routes all inter-node BAN traffic. Receives all detections, gait events, acoustic events, identification results, and recording notifications from all nodes.
- **External Network Gateway:** The **sole** point of external connectivity. WiFi (home network primary) and optional cellular SIM (remote access, emergency contact). No other node connects to any external network.
- **App Server:** Runs the embedded HTTP/WebSocket server for the SENTINEL-WEAR companion app. Serves REST API, WebSocket event stream, and media streams. Accessible via WiFi on local network or via cellular when remote access is configured.
- **Recording Manager:** Aggregates recording availability notifications from all nodes. Manages belt-local recording storage (SD card). Serves recordings to companion app. Generates legal export packages.
- **Sensing:** mmWave radar (downward, torso-level) for independent ground-plane and proximity coverage.
- **SLAM Processor (Linux SoM variant):** Runs 360° panorama stitching from curved pendant cameras; runs SLAM pipeline for dense world model.

---

## 2. Connectivity Architecture Detail

### 2.1 WiFi (Primary External Path)

- **Standard:** 802.11 a/b/g/n/ac/ax (depending on SoM)
- **Use:** Local network connectivity for companion app; high-bandwidth media streaming; SLAM data transfer
- **Not present on:** Any other node in the mesh

### 2.2 Cellular / SIM (Secondary External Path — Optional)

- **Purpose:** Remote access when away from home WiFi; emergency contact alert delivery; cellular-primary deployments (rural, mobile)
- **SIM types supported:** nano-SIM (physical) or eSIM (embedded)
- **Module variants (user-selected):**
  - Quectel EC21 (LTE Cat 1, UART): alerts and metadata, minimal data use
  - Quectel EC25 (LTE Cat 4, UART/USB): compressed video clips, adequate for most remote access
  - Quectel RM502Q-AE (5G Sub-6GHz, USB 3.x): highest bandwidth — live 360° relay possible
  - Quectel EG912Y-GL (LTE Cat 12, eSIM): multi-carrier, no physical SIM swap
- **Data path:** Cellular module connects only to the companion app (user-configured endpoint). No telemetry, no third-party connections.
- **Cellular configuration (`sentinel-wear.toml`):**
```toml
[connectivity.cellular]
enabled = false
sim_type = "nano_sim"            # "nano_sim" | "esim"
apn = "your.carrier.apn"
fallback_to_wifi = true          # Use WiFi when available; cellular as fallback
always_on_cellular = false       # Keep cellular always connected (higher battery draw)
alert_via_cellular = true        # Send critical alerts via cellular when active
stream_via_cellular = false      # Stream video/audio via cellular (data cost consideration)
emergency_contact = ""           # Phone number or endpoint for emergency alerts
emergency_contact_trigger = "manual"   # "manual" | "fall_detected" | "critical_alert"
```

### 2.3 Bluetooth (Direct Local — Fallback)

- BLE 5.3 direct connection to companion app (no WiFi required)
- Sufficient for: alerts, configuration, pairing, firmware updates
- **Not** sufficient for: video streaming, 360° panorama, SLAM data

### 2.4 Cellular Physical Hardware

The belt node PCB includes:
- nano-SIM card slot (standard 4FF form factor)
- OR eSIM footprint (depending on selected cellular module)
- Cellular module connected via UART (Cat 1/4) or USB 3.x (5G modules)
- RF antenna connector (U.FL) for external cellular antenna routed to enclosure edge
- GNSS capability (if cellular module includes GNSS) for emergency contact location

---

## 3. Compute Variants

### Variant A — MCU (Research Minimum)

- **MCU:** STM32H7 class (Cortex-M7, 480 MHz)
- Sufficient for: full body-frame fusion (6 nodes), PentaTrack, alert routing, BAN hub, basic recording management
- Companion app: WiFi via external WiFi module (ESP32 or similar co-processor)
- Cellular: Quectel EC21 via UART
- No SLAM, no 360° stitching
- ~2000 mAh battery

### Variant B — Linux SoM (Full Capability)

- **SoM:** Raspberry Pi Compute Module 4 (quad-core Cortex-A72, 4–8 GB RAM, built-in WiFi/BT)
- Full Linux, all crates including `sentinel-slam`, `sentinel-api` (axum HTTP server)
- Full companion app server: REST API, WebSocket, RTSP, H.264, 360° streams
- 360° panorama stitching from curved pendant cameras
- SLAM-ready: ORB-SLAM3 or LIO-SAM
- Cellular: Quectel EC25 or RM502Q via USB
- 5000–7000 mAh battery bank
- Requires thermal management

### Variant C — High-Performance SoM

- **SoM:** NXP i.MX 8M Plus (quad-core Cortex-A53 + Cortex-M7, NPU 2.3 TOPS)
- All Variant B capabilities
- NPU enables on-device neural classification
- M7 co-processor handles real-time BAN without interrupting A53 fusion
- 5G cellular module for live 360° streaming via cellular
- 5000–7000 mAh battery bank

### Variant D — Minimal (ESP32)

- **MCU:** ESP32-S3 (integrated WiFi + BLE)
- Lowest cost, sufficient for 2–3 nodes
- WiFi native to MCU
- Cellular: Quectel EC21 via UART (optional)
- ~2000 mAh

---

## 4. MCU Co-Processor Pattern (Variants B, C)

For Linux SoM variants, a co-processor handles real-time tasks:
- **Variant A co-proc:** STM32U5 (BAN radio, sensor I/O, real-time interrupts)
- **Variant B co-proc:** nRF5340 network core (BAN BLE 5.3 protocol, time-sync master for all nodes)

The co-processor runs `no_std` Rust firmware. The Linux SoM runs the full Rust crate stack as a native Linux process.

---

## 5. Hardware Interfaces Summary

| Interface | Purpose | Present on Belt Only? |
|---|---|---|
| WiFi (802.11) | Companion app, media streaming | **Yes — belt only** |
| Cellular SIM | Remote access, emergency contact | **Yes — belt only** |
| BLE 5.3 (BAN) | All inter-node communication | All nodes (BAN transport) |
| UWB (optional) | Precision time-sync + ranging | All nodes (optional) |
| USB-C | PC companion app + firmware update + charging | Yes |
| mmWave Radar (down) | Downward torso-level presence | Yes |
| IMU | Torso orientation (body-frame origin) | Yes |
| Environmental | Atmospheric context | Yes |
| SD card | Recording storage | Yes (primary storage) |
| LTE/5G antenna | Cellular RF | Yes |

---

## 6. Downward-Facing mmWave Radar

- **Variant A:** TI IWR6843AOP (60 GHz, AOA, oriented downward)
- **Variant B:** Acconeer XR112 (ultra-compact, lower power)

Coverage: ground-level objects approaching wearer's feet, surface for gait analysis cross-reference, trip hazards.

---

## 7. IMU (Torso Reference — Critical)

- **Variant A:** Bosch BMI270 (6-axis, 0.1 mg noise, consistent across all nodes)
- **Variant B:** TDK ICM-42688-P (0.07 mg noise, smallest package)

Belt IMU noise density is the most critical parameter in the system. All other node orientations express relative to this. Higher quality here benefits the entire body-frame model.

---

## 8. Power and Thermal

### Battery

- Variant A, D: Li-Ion 18650 × 1 (3500 mAh)
- Variant B, C: Li-Ion 18650 × 2 parallel (7000 mAh)
- Cellular module adds ~300 mA active draw; size battery accordingly

### Thermal (Linux SoM variants)

Sustained compute (SLAM, stitching, app server, cellular) generates 3–7 W. Requirements:
- Thermal pad under SoM footprint
- Vented or heatspreader enclosure
- NTC thermistor for thermal monitoring
- Firmware throttles SLAM frame rate above 50°C

---

## 9. No Identification Layer

The Belt Node does not carry an identification sensor. No camera, no biometric sensor, no hardware switch for identification purposes. Identification is the role of the Pendant Node (user-configured).

---

## 10. Form Factor

- **Belt buckle enclosure:** Replaces standard buckle. 80 mm × 60 mm × 20 mm.
- **Rear clip pack:** 100 mm × 70 mm × 25 mm. More internal volume.
- Weight: 100–300 g including battery bank (supported by hip, not jewelry-constrained).
- Charging: USB-C PD preferred.
- Cellular antenna: flush-mounted on enclosure edge or stubby external.

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
│   └── belt_node_bom.csv
└── README.md
```
