# Body-Coordinate Fusion — Architecture and Drift Correction

**Project:** SENTINEL-WEAR
**Domain:** Multi-node sensor fusion in non-rigid coordinate frames

## 1. The Problem

A wearable sensor mesh observes the world from multiple body-worn positions — pendant, bracelets, belt, anklets, optionally eyewear. Each position has its own local frame defined by the orientation of the body part it's attached to. The body itself is a non-rigid multi-segment frame: the head rotates relative to the torso, arms move relative to the torso, legs move relative to the torso, and the torso rotates and translates through the world.

A naive fusion that simply combines each node's local-frame detections produces a perception field that rotates when the wearer turns, lurches when the wearer walks, and reports phantom approaches whenever the wearer gestures. The fusion problem is to subtract wearer motion and produce a stable body-frame perception field — one in which "approach from the right rear" means an actual approach from the right rear, regardless of which way the wearer is facing.

This document specifies the architecture.

## 2. Coordinate Frames

Three frames matter:

- **World frame (W).** The fixed external reference frame.
- **Torso frame (T).** Origin at a chosen reference point on the wearer's torso (e.g., sternum), with axes aligned to the torso's anatomical orientation. This is SENTINEL-WEAR's primary perception frame.
- **Node frame (Nᵢ).** The local frame of node *i*, attached to the body part the node is worn on. Each node's IMU continuously reports its orientation, allowing transformation from Nᵢ to T.

The transform from Nᵢ to T is denoted ᵀTNᵢ and is updated every IMU sample. The transform from T to W is denoted ᵂTT and is the wearer's overall body orientation in the world.

## 3. Per-Node Transformation

Each node observes the world in its own local frame. A radar return at node *i* reports:

- *r*: range to the observed object.
- *φ*, *θ*: bearing in node *i*'s local frame.
- *v_r*: radial velocity in node *i*'s local frame.

To transform this observation into the torso frame:

1. Compute the object's position in node *i*'s frame from (*r*, *φ*, *θ*).
2. Apply ᵀTNᵢ to obtain the position in the torso frame.
3. Transform the radial-velocity component using the same rotation, plus the node's velocity relative to the torso (computed from the time derivative of ᵀTNᵢ).

The node-to-torso transform ᵀTNᵢ is computed from the relative orientation of the node's IMU and the torso reference IMU (typically located in the belt node). Translation between the node and the torso reference is small (centimeters) compared to typical observation ranges (meters), so position-translation contributions are usually negligible relative to angular contributions.

## 4. Drift Correction (Wearer Motion Subtraction)

Once observations are in the torso frame, the perception field is stable with respect to head and arm motion but still moves with the torso itself. If the wearer turns the torso, the world appears to rotate.

Body-frame drift correction subtracts this rotation by applying ᵂTT (the torso-to-world transform) to recover world-frame observations, then re-rotating into a *stabilized* torso frame that tracks the wearer's translational motion but not their rotational motion.

The choice of stabilization matters and is configurable:

- **Full world-frame stabilization.** "Approach from the north" is reported regardless of which way the wearer is facing. Useful for navigation and large-area awareness.
- **Translation-only stabilization (default).** Wearer rotation is subtracted, but wearer translation is preserved. "Approach from the right" means "approach from a direction that was the wearer's right at the time of observation, even if the wearer has since turned." Useful for threat awareness because it preserves the alert direction relative to the wearer's body axes at the moment of observation.
- **Full torso-frame (no stabilization).** The raw torso-frame perception, useful for diagnostics.

The default is translation-only stabilization, because it produces the most intuitive haptic-alert directionality for the wearer.

## 5. Multi-IMU Estimation

ᵂTT is estimated from the IMU stack distributed across the wearer's body. A single-IMU estimate (e.g., from the belt) drifts because of accelerometer bias and gyroscope drift; published research on multi-IMU body-pose estimation (the literature on inertial motion capture, OpenSim and similar biomechanical analysis tools, and consumer body-tracking products) provides the substrate for fusing multiple IMUs into a stable body-pose estimate.

The fusion architecture is:

- Each IMU produces a local orientation estimate via standard sensor fusion (Madgwick, Mahony, or extended Kalman filter).
- Pairwise constraints between adjacent body segments (anatomical kinematic constraints) regularize the joint estimate.
- Magnetometer input where available provides a heading reference; in indoor or magnetic-disturbance environments, magnetometer input is downweighted.
- Visual or other external references (when present) provide periodic absolute-orientation correction.

The result is a torso orientation estimate stable to within a few degrees over typical wear sessions.

## 6. PentaTrack Integration

PentaTrack operates in the stabilized torso frame. Each tracked object's predictive-center field is computed in this frame, drift analysis is computed against detections in this frame, and intercept projections (where applicable) are computed in this frame.

The key adaptation from PentaTrack's standard usage is that PentaTrack's "world frame" is, for SENTINEL-WEAR, the stabilized torso frame, not the actual world frame. PentaTrack's drift profiles for object types are correspondingly tuned for body-frame coordinates (a "human approaching" has a drift profile in the torso frame that differs from the same human's drift profile in the world frame).

## 7. Calibration

Initial calibration establishes the position and orientation of each node relative to the torso reference. The calibration procedure:

1. Wearer stands in a neutral pose (arms relaxed at sides, head facing forward, torso vertical).
2. The system records each node's orientation, designating these as the node's neutral orientation.
3. Wearer performs a sequence of standardized motions (turn head left/right, raise arms, etc.).
4. The system observes each node's orientation through the motion sequence and refines the node-to-torso transform.

Calibration takes approximately 60 seconds. Re-calibration is recommended after re-positioning a node (e.g., putting on a different bracelet) or after extended storage.

## 8. Failure Modes

Body-coordinate fusion can degrade in several conditions:

- **Severe magnetic disturbance.** Heading drift accumulates; the stabilized torso frame slowly rotates relative to the actual world. Translation-only stabilization is robust to this; world-frame stabilization is not.
- **Node loss.** If a node fails or is removed, the kinematic constraints involving that node are lost. The system reports degraded coverage for the affected body region.
- **Extreme motion.** Sustained accelerations beyond typical human motion (e.g., vehicle crash, free fall) violate the assumptions of the IMU fusion algorithms. The system flags these conditions and switches to a degraded perception mode.
- **Calibration drift.** Long wear sessions can accumulate calibration error. The system periodically reports an estimated calibration confidence and prompts re-calibration when the confidence drops below threshold.

## 9. Civilian Transfer

The body-coordinate-fusion architecture transfers directly to:

- **Sports biomechanics.** Multi-IMU body-pose estimation for technique analysis. Mature commercial space.
- **Accessibility wearables.** Stable directional haptic alerts for visually impaired users; the torso-frame stabilization is what makes the alerts intuitive.
- **Physical therapy and rehabilitation.** Motion-pattern characterization across multiple body segments.
- **Research-grade gait analysis.** Multi-IMU gait reconstruction.
- **Industrial wearables.** Workplace-safety wearables that need stable body-frame awareness independent of wearer head motion.

The fusion architecture is the same in each case; the perception-field consumer changes.
