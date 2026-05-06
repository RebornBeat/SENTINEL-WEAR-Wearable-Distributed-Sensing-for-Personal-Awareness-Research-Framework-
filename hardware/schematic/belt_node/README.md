# Belt Node — Primary Compute & Torso Reference

**Project:** SENTINEL-WEAR
**Node Position:** Waist belt / belt clip
**Role:** Primary compute, battery hub, mesh gateway, downward sensing, torso reference frame
**Status:** Reference Design

---

## 1. Role in the Mesh

The Belt Node is the central coordination unit of the SENTINEL-WEAR system. It acts as the **Body-Frame Origin** for the entire wearable mesh.

**Primary Functions:**
- **Compute Hub:** Runs the main fusion algorithms (`sentinel-fusion`), predictive tracking (`pentatrack_bridge`), and system coordination.
- **Torso Reference:** Provides the inertial reference frame for the body. The IMU on the belt node defines the "forward" and "up" vectors for the entire system. All other node orientations are expressed relative to this one.
- **Sensing:** Equipped with its own sensors for independent downward and forward torso-level coverage.
- **Communication Hub:** Manages the Body-Area Network (BAN), routing data from all other nodes.

---

## 2. Candidate Component Set (Test Variants — Not Locked)

### Primary Compute Module

- **Variant A:** Raspberry Pi Compute Module 4 (CM4) — quad-core Cortex-A72, 4–8 GB RAM, eMMC, Wi-Fi, BT. Runs full Linux + `sentinel-belt-controller` binary. Maximum capability.
- **Variant B:** NXP i.MX 8M Plus — quad-core Cortex-A53 + Cortex-M7, NPU 2.3 TOPS, Linux/RTOS. Enables on-device neural inference for classification enhancement.
- **Variant C:** Qualcomm SA8155P — automotive-grade SoC, Linux, high compute for real-time body-frame fusion at scale.
- **Variant D:** ESP32-S3 + dedicated MCU for sensor handling — lower cost, embedded RTOS, limited classification capability. Suitable for minimal-node deployments.

**Selection criteria:** Ability to run Rust `sentinel-belt-controller` binary, RAM sufficient for full body-frame fusion (PentaTrack + 6 nodes), BAN radio integration, battery life impact.

### MCU Co-Processor (Sensor Interface & Real-Time)

- **Variant A:** STM32U5 (handles sensor I/O, BAN radio, while primary compute handles fusion)
- **Variant B:** nRF5340 network core (handles BAN protocol while app core handles fusion)

A co-processor pattern separates real-time sensor interrupt handling (sub-millisecond) from OS-level fusion processing. This is required when the primary compute runs a full OS with non-deterministic scheduling.

### Downward-Facing mmWave Radar

- **Variant A:** TI IWR6843AOP (60 GHz, AOA capable, oriented downward from belt buckle)
- **Variant B:** Acconeer XR112 (ultra-compact, lower power)

**Coverage:** Detects objects approaching from below knee level, ground surface for gait analysis (combined with anklet IMUs), potential trip hazards.

### IMU (Torso Reference — Most Critical)

- **Variant A:** Bosch BMI270 (consistent with all other nodes)

The belt IMU is the torso reference. All other node IMUs are expressed relative to this one in the body-frame coordinate system. Bias stability and noise density are the primary selection criteria.

### BAN Radio (Body-Area Network Hub)

- **Variant A:** Nordic nRF52840 co-processor (BLE 5.3 mesh, handles all inter-node communication)
- **Variant B:** Decawave DW3000 UWB (for precise ranging and sub-ms sync — research variant)

### External Interface

| Interface | Purpose |
|---|---|
| Wi-Fi 6 (via compute module) | Companion app connection on home network |
| Bluetooth 5.3 | Mobile companion app (when off home network) |
| USB-C | PC companion app + charging |
| LTE module (optional) | Emergency contact when off home network |

**LTE Variant A:** Quectel EC21 (LTE Cat 1, UART, widely available)

### Battery

- Li-Ion 18650 cells (2× parallel for 5000–7000 mAh) — primary battery bank.
- Powers: all belt-node compute + charges bracelet/anklet nodes via integrated charging ICs.
- Target 8–24 hours runtime depending on activity level and sensing configuration.

---

## 3. Hardware Architecture Block Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         Belt Node                           │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────┐   │
│  │ mmWave Radar│  │  IMU        │  │ Environmental Sens│   │
│  │ (downward)  │  │ (Torso Ref) │  │ (Temp/Press/VOC)  │   │
│  └──────┬──────┘  └──────┬──────┘  └─────────┬─────────┘   │
│         └────────────────┼──────────────────── ┘            │
│                          │                                  │
│                   ┌──────▼──────┐                           │
│                   │  MCU Co-Proc│ (Real-time sensor I/O)    │
│                   │  (nRF52840  │  + BAN Hub                │
│                   │   / STM32U5)│                           │
│                   └──────┬──────┘                           │
│                          │ IPC (SPI/UART)                   │
│                   ┌──────▼──────┐                           │
│                   │ Primary     │ (Fusion, Tracking,        │
│                   │ Compute SoC │  System Coordination)     │
│                   │ (CM4 / iMX8)│                           │
│                   └──────┬──────┘                           │
│                          │                                  │
│         ┌────────────────┼────────────┐                     │
│         │                │            │                     │
│  ┌──────▼──┐     ┌───────▼──┐  ┌─────▼────┐                │
│  │ Wi-Fi / │     │  USB-C   │  │ Battery  │                │
│  │  BLE /  │     │ Debug /  │  │ Mgmt     │                │
│  │   LTE   │     │  Charge  │  │ (PMIC)   │                │
│  └─────────┘     └──────────┘  └──────────┘                │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                 BAN Network (to all other nodes)
```

---

## 4. No Identification Layer

The Belt Node does **not** carry an identification sensor (camera or biometric). Identification is the role of the Pendant Node (opt-in only). No hardware kill-switch for an identification sensor is required on this node.

---

## 5. Power Management

- **Battery:** Largest capacity in the system (worn at waist, hip support).
- **Charging:** USB-C input (PD preferred for faster charging).
- **Thermal:** Compute load generates heat. Enclosure must dissipate heat away from the body. Thermal vias under compute module; vented enclosure panels.
- **Research Question:** Can the belt node power other nodes inductively? (Future iteration — not in scope for v1).

---

## 6. Form Factor

- **Enclosure:** Must be comfortable for continuous wear at the waist.
- **Weight:** Higher tolerance than other nodes (up to ~200–300 g including battery bank).
- **Attachment:** Belt clip or integrated into a belt buckle module.

---

## 7. Design Variants

| Variant | Processor | Primary Use Case |
|---|---|---|
| **Standard** | High-perf MCU (STM32H7) | Runs `sentinel-belt-controller` Rust binary, embedded RTOS |
| **Advanced** | Linux SoM (CM4, iMX8) | Full Linux stack, heavy fusion models, companion app server |
| **Minimal** | ESP32-S3 | Low-cost single-chip, minimal classification capability |

---

## 8. Interface Summary

| Interface | Function | Notes |
|---|---|---|
| **BAN Radio (Hub)** | Mesh network coordinator | All nodes relay through belt |
| **USB-C** | Power + Debug | Charging + firmware flashing |
| **mmWave Radar** | Presence/velocity | Downward-facing torso-level |
| **IMU** | Torso orientation | **Critical: defines body frame** |
| **Environmental Sensor** | Context data | Temp/Pressure optional |
| **Wi-Fi / BLE** | Companion app | Home network + mobile |
| **LTE** | Emergency contact | Optional module |

---

## 9. Directory Structure

```
hardware/schematic/belt_node/
├── belt_node.kicad_pro
├── belt_node.kicad_sch
├── belt_node.kicad_pcb
├── belt_node.kicad_prl
├── fp-info-cache
├── fp-lib-table
├── sym-lib-table
└── README.md
```
