# Eyewear Node — Hardware Schematic Reference Design

**Node Position:** Face / glasses frame
**Sensor Coverage:** Forward hemisphere, head-stabilized
**Status:** Optional node. Reference design, variant 1 of N.

---

## Purpose

The eyewear node provides head-stabilized forward-hemisphere sensing. Because the head tracks the wearer's gaze, the eyewear node faces whatever the wearer is looking at — providing the most precise forward coverage of any node in the mesh.

This node is optional (the system operates without it). It adds value for:
- High-precision forward detection (ahead of the wearer's gaze direction)
- Fast-moving object detection via event-based vision (the most natural position for event cameras)
- Audio event localization in the forward direction

---

## Candidate Component Set (Test Variants — Not Locked)

### Form Factor Variants
The eyewear node can be implemented as:
- **Variant A:** Integrated into glasses frame (both temporal arms carry PCBs)
- **Variant B:** Clip-on module for existing glasses (single PCB, clips to nose bridge area)
- **Variant C:** Standalone headband (for users without glasses; elastic headband with PCB in center forehead position)

### MCU
- **Variant A:** Nordic nRF5340 (as other nodes)
- **Variant B:** STM32L4 (ultra-low power for glasses — weight and battery are most critical)

### Event Camera (Primary Sensor — Head Position Ideal)
The eyewear position is ideal for an event camera: head-stabilized, no body-motion noise, facing the direction of attention.
- **Variant A:** Prophesee Metavision EVK3 (640×480, USB, evaluating miniaturization)
- **Variant B:** Custom ASIC from Prophesee or iniVation at reduced resolution for wearable form factor
- **Variant C:** Standard DVS sensor (Dynamic Vision Sensor) from iniVation at HVGA

Selection criteria: physical size compatible with glasses frame (PCB ≤ 15 mm × 10 mm for frame-integrated variants), power consumption (glasses battery severely constrained), resolution for fast-object detection at body-frame distances.

### IMU
- **Variant A:** Bosch BMI270 (head-segment IMU — contributes head-to-torso relative orientation for body-frame computation)

### BAN Radio
- BLE 5.3 via MCU

### Battery
- LiPo 3.7V, 50–80 mAh (most constrained node — glasses can only accommodate a very small battery)
- Expected runtime: 4–8 hours per charge (significantly less than other nodes due to battery constraint)

---

## Design Tradeoffs

**The key challenge** for eyewear nodes is the extreme size and weight constraint. A standard glasses arm is approximately 5 mm wide × 2 mm thick. A PCB that fits in both arms of a glasses frame is approximately 70 mm × 3 mm × 1 mm for each arm — extremely challenging.

Initial testing should use the clip-on variant (single PCB, 30 mm × 20 mm, 5 g) to validate sensor performance before attempting frame-integrated miniaturization.

**The goal is not a production-spec design on the first iteration.** The goal is to determine: (a) what sensing value the eyewear position adds, (b) what sensor configuration achieves that value, and (c) what the miniaturization target must be for a form-factor-acceptable implementation. These questions are answered by testing the clip-on variant first.
