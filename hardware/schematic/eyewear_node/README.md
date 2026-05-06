# Eyewear Node — Hardware Schematic Reference Design

**Project:** SENTINEL-WEAR
**Node Position:** Face / glasses frame
**Primary Role:** Forward hemisphere fast-event detection, head-pose estimation (optional node)
**Status:** Optional node. Reference Design.

---

## 1. Purpose

The Eyewear Node provides a forward-facing, head-stabilized sensing perspective in the SENTINEL-WEAR body-frame mesh. Because the head tracks the wearer's gaze, the eyewear node faces whatever the wearer is looking at — providing the most precise forward coverage of any node in the mesh.

This node is optional (the system operates without it). It adds value for:
- **High-precision forward detection** (ahead of the wearer's gaze direction).
- **Fast-moving object detection** via event-based vision — the most natural position for event cameras (microsecond-latency response).
- **Head-pose estimation** — the IMU here enables the system to separate head rotation from torso rotation, significantly improving body-frame stabilization accuracy.
- **Audio event localization** in the forward direction.

In the body-frame fusion architecture, the Eyewear Node is the primary source for:
- Event-stream analysis (fast transients in the forward view).
- Head-pose estimation (via IMU fusion with the belt node's torso reference).
- Optional Identification Layer (if a conventional camera is installed).

---

## 2. Architecture: Limitless Configuration

SENTINEL-WEAR imposes no artificial constraints on the hardware design. The Eyewear Node is a **configurable platform**, not a fixed product.

### 2.1 Sensor Configuration

**Baseline (minimum) sensor stack:**
- **Event-based vision sensor:** Primary modality. No fixed resolution, FoV, or latency constraint. User selects sensor fitting the form-factor and power budget. Event sensors are inherently low-latency; this position is ideal for them.
- **IMU (6-axis or 9-axis):** For head-pose estimation.

**Optional sensors (research-driven, no constraints):**
- **Environmental sensor:** Temperature, humidity (for atmospheric correction of forward-facing sensors).
- **Bone-conduction actuator:** For directional alerts in the forward hemisphere (alternative to audio alerts, maintains situational awareness).

**Optional Identification Layer:**

If a conventional camera is added to the Eyewear Node:
- It becomes part of the **Identification Layer**.
- It is **not part of the sensing mesh**. It does not track motion.
- It is triggered exclusively by the **Policy Engine**.
- **Hardware Kill Switch is required** — must physically cut power and data to the imaging sensor.
- The MCU must verify kill-switch state at boot.

**Kill Switch:**
If an Identification sensor is present:
- Must physically cut power or data to the sensor.
- MCU must read GPIO state at boot and halt if DISABLED.
- No firmware bypass permitted.

### 2.2 Connectivity

- **BAN Transport Primary:** BLE 5.x via MCU integrated radio.
- **BAN Transport Optional:** UWB (Qorvo DW3000) for lower latency.
- **Body-coupled communication** (research track).
- **Data transmitted:** Processed metadata only. Raw event streams can be logged locally if configured, but are not transmitted to preserve BAN bandwidth.

### 2.3 Power

- **Configurable power modes:**
  - Low-power polling: Event sensor on standby, IMU at low rate.
  - Active mode: Event sensor active for fast-event detection.
  - High-power mode: Identification camera active (if present and policy-allowed).
- **Charging:** Standard Qi wireless or magnetic pogo pins.
- No constraint on battery chemistry or capacity — user determines form-factor vs. runtime tradeoff.

---

## 3. Candidate Component Set (Test Variants — Not Locked)

### Form Factor Variants

- **Variant A:** Integrated into glasses frame (both temporal arms carry PCBs)
- **Variant B:** Clip-on module for existing glasses (single PCB, clips to nose bridge area) — **recommended for initial validation**
- **Variant C:** Standalone headband (for users without glasses; elastic headband with PCB in center forehead position)

### MCU

- **Variant A:** Nordic nRF5340 (consistent with other nodes)
- **Variant B:** STM32L4 (ultra-low power — weight and battery are most critical in glasses form factor)

### Event Camera (Primary Sensor)

- **Variant A:** Prophesee Metavision EVK3 (640×480, USB, lower power) — evaluation/research variant
- **Variant B:** Custom ASIC from Prophesee or iniVation at reduced resolution for wearable form factor
- **Variant C:** iniVation DAVIS346 (346×260, USB, combined frame+events)

**Selection criteria:** Physical size compatible with glasses frame (PCB ≤ 15 mm × 10 mm for frame-integrated variants), power consumption (glasses battery severely constrained), resolution for fast-object detection at body-frame distances.

### IMU

- **Variant A:** Bosch BMI270 (head-segment IMU — contributes head-to-torso relative orientation for body-frame computation)
- **Variant B:** Invensense ICM-42688-P (smallest package, for frame-integrated space constraint)

### BAN Radio

- BLE 5.3 via MCU integrated radio

### Battery

- LiPo 3.7V, 50–80 mAh (most constrained node — glasses can only accommodate a very small battery)
- Expected runtime: 4–8 hours per charge (significantly less than other nodes due to battery constraint)

---

## 4. Form Factor & Human Factors

### 4.1 Physical Design

Integrates into standard eyewear forms: prescription glasses, safety glasses, tactical eyewear, sport eyewear.

Must:
- Preserve wearer's field of view entirely.
- Avoid visual obstruction.
- Maintain balanced weight distribution left/right.
- Be compatible with prescription lenses (mounting solution accounts for lens thickness).

### 4.2 Weight Target

- Jewelry-grade or lighter.
- User-configurable: some users accept heavier frames for longer battery; others prioritize minimalism.

### 4.3 Durability

- Minimum: Sweat resistance (IPX4).
- Rain resistance for outdoor use.
- IP67 for rugged/tactical variants.

### 4.4 Skin Contact

- All materials must be hypoallergenic.
- Nose pads and temple tips must meet EN 1811 for nickel release on prolonged skin contact.

---

## 5. Design Tradeoffs

**The key challenge** for eyewear nodes is the extreme size and weight constraint. A standard glasses arm is approximately 5 mm wide × 2 mm thick. A PCB that fits in both arms is approximately 70 mm × 3 mm × 1 mm for each arm — extremely challenging at board assembly scale.

**Initial testing strategy:** Use the clip-on variant (single PCB, 30 mm × 20 mm, ~5 g) to validate sensor performance before attempting frame-integrated miniaturization. This answers:
1. What sensing value does the eyewear position add?
2. What sensor configuration achieves that value?
3. What is the miniaturization target for a form-factor-acceptable implementation?

The goal of the first iteration is to answer these research questions, not to produce a finished product.

---

## 6. Implementation Notes

### Event Sensor Integration

Event-based sensors (Prophesee, iniVation) are the preferred modality because:
- Microsecond-scale response times.
- Highly sensitive to motion in the forward field.
- Sparse data output, reducing BAN bandwidth.

No fixed resolution is enforced. Sensor selected based on research requirement.

### IMU Fusion with Belt Node

The eyewear IMU is fused with the belt node's IMU to provide:
- Head-pose estimation.
- Separation of head vs. torso rotation.

This significantly improves body-frame stabilization accuracy. Without the eyewear IMU, the system assumes head and torso are aligned.

### Identification Layer Integration (If Present)

Triggered exclusively by the Policy Engine. The Policy Engine evaluates:
- Fast-object detection by the event sensor.
- Unknown person approaching (detected by other nodes).
- Manual trigger by wearer.

Respects Policy Engine mode:
- **Privacy-First mode:** Camera never activates.
- **Security-First mode:** Camera auto-activates when trigger conditions are met.

---

## 7. Directory Structure

```
hardware/schematic/eyewear_node/
├── README.md
├── eyewear_node.kicad_pro
├── eyewear_node.kicad_sch
├── eyewear_node.kicad_pcb
├── gerbers/
│   └── (manufacturing files)
├── bom/
│   └── eyewear_node_bom.csv
├── assembly/
│   └── eyewear_node_pick_and_place.csv
├── 3d_models/
│   ├── frame_mount.step
│   └── temple_integration.step
├── datasheets/
│   └── (component datasheets)
└── config/
    └── hardware_config.md
```
