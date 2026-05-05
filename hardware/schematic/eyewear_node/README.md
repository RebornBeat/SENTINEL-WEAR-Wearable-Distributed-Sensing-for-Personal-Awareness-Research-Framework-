# Eyewear Node — SENTINEL-WEAR Hardware Reference

**Project:** SENTINEL-WEAR
**Node Type:** Optional / Specialized
**Primary Role:** Forward hemisphere fast-event detection, head-pose estimation

---

## 1. Purpose

The Eyewear Node provides a forward-facing, head-stabilized sensing perspective in the SENTINEL-WEAR body-frame mesh. It is an optional node designed to extend coverage into the wearer’s direct line of sight and to provide high-quality head-pose data that significantly improves body-frame fusion accuracy.

Unlike the Pendant Node (which covers the upper torso hemisphere) or the Bracelet Nodes (which cover the lateral body volume), the Eyewear Node offers:

- **Direct line-of-sight alignment:** Sensors point where the wearer is looking.
- **Head-pose independence:** The IMU on the eyewear allows the system to separate head rotation from torso rotation, improving the accuracy of body-frame stabilization.
- **Fast-event detection:** The primary sensor is an **event-based vision sensor**, enabling microsecond-latency detection of fast-moving objects (relevant to the extreme-velocity sensing research track).

**In the body-frame fusion architecture:**
The Eyewear Node is the primary source for:
- **Event-stream analysis** (fast transients in the forward view).
- **Head-pose estimation** (via IMU fusion with the belt node’s torso reference).
- **Optional Identification Layer** (if a conventional camera is installed).

---

## 2. Architecture: Limitless Configuration

SENTINEL-WEAR imposes no artificial constraints on the hardware design. The Eyewear Node is a **configurable platform**, not a fixed product.

### 2.1 Sensor Configuration

**Baseline sensor stack:**
- **Event-based vision sensor:** Primary modality.
  - No fixed resolution constraint. User selects the sensor that fits the form-factor and power budget.
  - No fixed field-of-view constraint.
  - No fixed latency constraint (event sensors are inherently low-latency).
- **IMU (6-axis or 9-axis):** For head-pose estimation.

**Optional sensors (research-driven):**
- **Environmental sensor:** Temperature, humidity (for atmospheric correction).
- **Bone-conduction actuator:** For directional alerts in the forward hemisphere (optional alternative to audio alerts).

**Optional Identification Layer:**
If a conventional camera is added:
- It becomes an **Identification Node**.
- It falls under the **Identification Layer Policy**:
  - Must be opt-in via configuration.
  - Must respect the user’s **Privacy-First / Security-First** mode.
  - If the Policy Engine is in **Privacy-First** mode, the Identification module is electrically isolated or driver-disabled.
  - If in **Security-First** mode, it can auto-activate when the Policy Engine detects a trigger event (e.g., fast-moving object detected by the event sensor).

**Kill Switch:**
If an Identification sensor is present, a **hardware kill switch** is required:
- Must physically cut power or data to the sensor.
- System firmware must verify switch state at boot.

### 2.2 Connectivity

The Eyewear Node communicates with the Belt Node (primary compute) over the **Body-Area Network (BAN)**.

**Transport options:**
- **BLE 5.x** (default).
- **UWB** (optional, for lower latency).
- **Body-coupled communication** (research).

**Architecture decision:**
- The node transmits **processed metadata**, not raw event streams.
- Raw event streams can be logged locally on the node if configured, but are not transmitted to preserve BAN bandwidth.
- The user configures what data leaves the node.

### 2.3 Power

**Target:**
- Full-day operation under typical usage.
- Configurable power modes:
  - **Low-power polling:** Event sensor on standby, IMU at low rate.
  - **Active mode:** Event sensor active for fast-event detection.
  - **High-power mode:** Identification camera active (if present and policy-allowed).

**Charging:**
- Standard wireless charging (Qi-compatible) or magnetic pogo pins.
- No constraint on battery chemistry or capacity—user determines acceptable form-factor vs. battery life trade-off.

---

## 3. Form Factor & Human Factors

### 3.1 Physical Design

The Eyewear Node is designed to integrate into standard eyewear forms:
- **Glasses frames.**
- **Safety glasses.**
- **Tactical eyewear.**
- **Sport eyewear.**

The design must:
- Preserve the wearer’s field of view.
- Avoid creating visual obstruction.
- Maintain balance (weight distribution left/right).
- Be compatible with prescription lenses (mounting solution must account for lens thickness).

### 3.2 Weight

**Target:**
- Jewelry-grade or lighter.
- User-configurable: some users accept heavier frames for longer battery life; others prioritize minimalism.

### 3.3 Durability

**Minimum:**
- Sweat resistance (IPX4).
- Rain resistance for outdoor use.

**User-configurable:**
- IP67 or higher for rugged/tactical variants.

### 3.4 Skin Contact

- Materials must be hypoallergenic.
- Nose pads and temple tips must meet standards for prolonged skin contact (e.g., EN 1811 for nickel release).

---

## 4. Directory Structure

The Eyewear Node hardware is organized as follows:

```
hardware/eyewear_node/
├── README.md                 # This document
├── kicad/
│   ├── eyewear_node.kicad_pro
│   ├── eyewear_node.kicad_sch
│   └── eyewear_node.kicad_pcb
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
    └── hardware_config.md    # Pin mappings, I2C addresses, power domains
```

---

## 5. Implementation Notes

### 5.1 Event Sensor

Event-based sensors (e.g., Prophesee, IniVation) are the preferred modality because:
- They provide microsecond-scale response times.
- They are highly sensitive to motion in the forward field.
- They produce sparse data, reducing bandwidth.

**No fixed resolution:**
- The system does not enforce a minimum or maximum resolution.
- The user selects the sensor based on the research requirement.

### 5.2 IMU Integration

The IMU on the Eyewear Node is fused with the Belt Node’s IMU to provide:
- **Head-pose estimation.**
- **Separation of head vs. torso rotation.**

This significantly improves the accuracy of the **body-frame stabilization** algorithm, which by default uses the **Translational stabilization mode** (frame co-translates with the wearer but does not co-rotate).

### 5.3 Identification Layer Integration

If a conventional camera is added to the Eyewear Node:
- It becomes part of the **Identification Layer**.
- It is **not part of the sensing mesh**.
- It does not track motion.
- It is triggered exclusively by the **Policy Engine**.

The Policy Engine evaluates conditions such as:
- **Fast-object detection** by the event sensor.
- **Unknown person approaching** (detected by other nodes).
- **Manual trigger** by the wearer.

The Policy Engine respects:
- **Privacy-First mode:** Camera never activates.
- **Security-First mode:** Camera auto-activates when trigger conditions are met.

---

## 6. Research & Experimentation

This node is a **research platform**, not a finished product.

- **No artificial limits.** Resolution, power, connectivity, and storage are fully configurable.
- **Datasets.** Raw event streams, IMU data, and identification images (if enabled) can be logged to local storage for later analysis.
- **Extensibility.** The design supports addition of sensors, alternative radios, and different power sources.

The eyewear node is where **fast perception** meets **body-frame context**. It is the optimal node for the extreme-velocity sensing research track, as it offers the most direct forward-facing perspective with the least occlusion.

---

## 7. Disclaimer

This hardware is for **research and education purposes**. It is not a medical device, not PPE, and not a certified safety product. Use at your own risk.
