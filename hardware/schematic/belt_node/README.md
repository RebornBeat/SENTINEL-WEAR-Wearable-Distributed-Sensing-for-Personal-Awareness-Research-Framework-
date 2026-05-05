# Belt Node — Hardware Schematic Reference Design

**Node Position:** Waist belt / belt clip
**Role:** Primary compute, battery hub, mesh gateway, downward sensing
**Status:** Reference design, variant 1 of N.

---

## Purpose

The belt node is the most capable node in the mesh. It runs the `sentinel-belt-controller` binary (the main fusion, tracking, and alerting process) and serves as the gateway between the body-area mesh and any external interfaces (phone, companion app, emergency contact). Its waist position also provides downward coverage of the ground plane.

Weight budget for belt nodes is relaxed (~100–300 g including battery) since belt-mounted devices are supported by the hip rather than the wrist or neck.

---

## Candidate Component Set (Test Variants — Not Locked)

### Primary Compute Module
- **Variant A:** Raspberry Pi Compute Module 4 (CM4) — quad-core Cortex-A72, 4–8 GB RAM, eMMC, Wi-Fi, BT. Runs full Linux + `sentinel-belt-controller` binary. Maximum capability.
- **Variant B:** NXP i.MX 8M Plus — quad-core Cortex-A53 + Cortex-M7, NPU 2.3 TOPS, Linux/RTOS. Enables on-device neural inference for classification enhancement.
- **Variant C:** Qualcomm SA8155P — automotive-grade SoC, Linux, high compute for real-time body-frame fusion at scale.
- **Variant D:** ESP32-S3 + dedicated MCU for sensor handling — lower cost, embedded RTOS, limited classification capability. Suitable for minimal-node deployments.

Selection criteria: ability to run Rust `sentinel-belt-controller` binary, RAM sufficient for full body-frame fusion (PentaTrack + 6 nodes), BAN radio integration, battery life impact.

### MCU Co-Processor (Sensor Interface)
- **Variant A:** STM32U5 (handles sensor I/O, BAN radio, while primary compute handles fusion)
- **Variant B:** nRF5340 network core (handles BAN protocol while app core handles fusion)

A co-processor pattern separates real-time sensor interrupt handling from OS-level fusion processing.

### Downward-Facing mmWave Radar
- **Variant A:** TI IWR6843AOP (60 GHz, AOA capable, oriented downward from belt buckle)
- **Variant B:** Acconeer XR112 (ultra-compact, lower power)

Downward coverage detects: objects approaching from below knee level, ground surface for gait analysis (combined with anklet IMUs), potential trip hazards.

### IMU (Torso Reference)
- **Variant A:** Bosch BMI270 (consistent with all other nodes)

The belt IMU is the torso reference. All other node IMUs are expressed relative to this one in the body-frame coordinate system.

### BAN Radio (Body-Area Network)
- **Variant A:** Nordic nRF52840 co-processor (BLE 5.3 mesh, handles all inter-node communication)
- **Variant B:** Decawave DW3000 UWB (for precise ranging and sub-ms sync — research variant)

### External Interface
- Wi-Fi 6 (via compute module) for companion app connection on home network
- Bluetooth 5.3 for mobile companion app (when not on same Wi-Fi)
- USB-C for PC companion app and charging
- LTE module (optional) for emergency contact when off home network — **Variant A:** Quectel EC21 (LTE Cat 1)

### Battery
- Li-Ion 18650 cells (2× parallel for 5000–7000 mAh) — primary battery bank
- Powers: all belt-node compute + charges bracelet/anklet nodes via integrated charging ICs
- Target 8–24 hours runtime depending on activity level and sensing configuration
