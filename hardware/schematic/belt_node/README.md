# Belt Node — Primary Compute & Torso Reference

**Project:** SENTINEL-WEAR
**Node Type:** Primary Compute Hub
**Form Factor:** Belt buckle / Belt-worn enclosure

---

## 1. Role in the Mesh

The Belt Node is the central coordination unit of the SENTINEL-WEAR system. It acts as the **Body-Frame Origin** for the entire wearable mesh.

**Primary Functions:**
- **Compute Hub:** Runs the main fusion algorithms (`sentinel-fusion`), predictive tracking (`pentatrack_bridge`), and system coordination.
- **Torso Reference:** Provides the inertial reference frame for the body. The IMU on the belt node defines the "forward" and "up" vectors for the entire system.
- **Sensing:** Equipped with its own sensors for independent coverage of the torso volume.
- **Communication Hub:** Manages the Body-Area Network (BAN), routing data from all other nodes (pendant, bracelets, anklets).

---

## 2. Sensor Stack

The Belt Node is a fully capable sensing node in addition to being the compute hub.

**Standard Suite:**
- **mmWave Radar:** Primary sensor for torso-level presence/velocity.
- **IMU (6/9-DOF):** Torso-frame orientation reference.
- **Environmental Sensors:** Temperature, pressure (optional).

**Optional / Research Variants:**
- **Solid-State LiDAR:** Short-range geometry (forward-facing).
- **Microphone Array:** Audio event detection at waist level.
- **Haptic Actuator:** For direct feedback.

**No Identification Layer:**
- The Belt Node does **not** carry an identification sensor (camera). Identification is the role of the Pendant Node (opt-in).

---

## 3. Hardware Architecture (Block Diagram)

```
                 ┌───────────────────────────────────────────┐
                 │               Belt Node                   │
                 │                                           │
                 │  ┌─────────┐  ┌─────────┐  ┌───────────┐  │
                 │  │ mmWave │  │   IMU   │  │ Env Sensor│  │
                 │  │ Radar  │  │(Torso)  │  │  (Opt)    │  │
                 │  └────┬────┘  └────┬────┘  └─────┬─────┘  │
                 │       │            │             │        │
                 │       └────────────┼─────────────┘        │
                 │                    │                      │
                 │              ┌─────┴─────┐                │
                 │              │   MCU /   │                │
                 │              │ App Proc  │                │
                 │              │           │                │
                 │              │ (Runs     │                │
                 │              │ sentinel- │                │
                 │              │ belt-     │                │
                 │              │ controller)│               │
                 │              └─────┬─────┘                │
                 │                    │                      │
                 │       ┌────────────┼────────────┐         │
                 │       │            │            │         │
                 │  ┌────┴────┐  ┌────┴────┐  ┌───┴───┐     │
                 │  │ BAN     │  │ USB-C   │  │ Batt  │     │
                 │  │ Radio   │  │ Debug/  │  │ Mgmt  │     │
                 │  │ (Hub)   │  │ Charge  │  │       │     │
                 │  └─────────┘  └─────────┘  └───────┘     │
                 │                                           │
                 └───────────────────┬───────────────────────┘
                                     │
                          BAN Network (to other nodes)
```

---

## 4. Processor Options

**Research Phase:** No single processor is mandated. The hardware design should be modular to accept different MCUs/SoCs based on research needs.

- **High-Performance MCU:** STM32H7, i.MX RT, or similar. Required for running the full `sentinel-belt-controller` Rust stack with `std` support.
- **Embedded Linux Module:** For maximum compute power (e.g., running complex models).

**Constraint:** Must have sufficient I/O for:
- 1x SPI/UART for mmWave Radar
- 1x I2C for IMU + Env Sensors
- 1x SPI/I2C for BAN Radio (BLE/UWB)

---

## 5. Power Management

- **Battery:** Largest capacity in the system (worn at waist).
- **Charging:** USB-C input.
- **Power Budget:** Must support continuous operation of the primary compute loop + BAN hub duties.
- **Research Question:** Can the belt node power other nodes inductively? (Out of scope for current version, but relevant for future iterations).

---

## 6. Form Factor

- **Enclosure:** Must be comfortable for continuous wear at the waist.
- **Weight:** Higher tolerance than other nodes (up to ~200g).
- **Thermal:** Compute load generates heat. Enclosure must dissipate heat away from the body.

---

## 7. Design Variants

| Variant | Processor | Role |
| :--- | :--- | :--- |
| **Standard** | High-perf MCU | Runs `sentinel-belt-controller` (Rust binary) |
| **Advanced** | Linux SoM | Runs full Linux stack (heavy fusion models) |

---

## 8. Directory Structure

```
hardware/schematic/belt_node/
├── belt_node.kicad_pro          # KiCad project file
├── belt_node.kicad_sch          # Main schematic
├── belt_node.kicad_pcb          # PCB layout
├── belt_node.kicad_prl          # Project settings
├── fp-info-cache                # Footprint cache
├── fp-lib-table                 # Footprint libraries
├── sym-lib-table                # Symbol libraries
└── README.md                    # This file
```

---

## 9. Interface Summary

| Interface | Function | Notes |
| :--- | :--- | :--- |
| **BAN Radio** | Mesh Network | Hub for all other nodes |
| **USB-C** | Power + Debug | Charging & firmware flashing |
| **mmWave Radar** | Presence/Velocity | Torso-level sensing |
| **IMU** | Orientation | **Critical:** Defines body frame |
| **Env Sensor** | Context | Temp/Pressure (optional) |

---

## 10. Future Research Directions

- **Inductive Charging:** Remove USB port for waterproofing.
- **Kinetic Charging:** Harvest energy from walking motion.
- **Expandable I/O:** Header for additional research sensors.
