# Body-Frame Calibration — SENTINEL-WEAR

---

## Overview

SENTINEL-WEAR's body-frame coordinate system requires two calibration steps before the system can produce meaningful directional detections:

1. **Neutral-pose calibration:** Establishes each node's baseline orientation relative to the body frame.
2. **Walk-through calibration:** Estimates each node's position on the body (where on the wrist, where on the ankle, etc.) using motion as a reference signal.

Both calibrations together allow the belt controller to maintain a stabilized body frame and transform detections from each node's local frame into unified body-frame coordinates.

---

## Neutral-Pose Calibration

### What it establishes

The anatomical alignment of each node's IMU relative to the torso reference. Without this, the system cannot distinguish "the left bracelet is rotated 45° relative to the torso" from "the torso itself rotated 45°."

### Procedure

1. Put on all nodes. Ensure they are positioned at their intended locations (pendant on chest, bracelets on wrists, belt around waist, anklets above ankles, eyewear on face).

2. Stand in the neutral anatomical pose:
   - Feet hip-width apart, weight balanced
   - Arms hanging naturally at sides (not crossed, not in pockets)
   - Head facing forward, level (not tilted)
   - Torso upright, not slouched

3. Hold this pose for 5 seconds without movement.

4. Trigger calibration:
```bash
curl -X POST http://belt-node.local/api/calibration/neutral-pose/start
# System records for 5 seconds
# Returns: {"status": "Collecting", "progress": 0}
# After 5 seconds:
curl http://belt-node.local/api/calibration/neutral-pose/status
# Returns: {"status": "Complete", "confidence": 0.97}
```

### What the system records

Each node's IMU Madgwick filter state at the neutral pose becomes the **reference orientation** for that node. The relative quaternion between each node and the belt-node IMU is stored as the node's body-frame mounting transform.

### When to redo

Redo neutral-pose calibration when:
- A node has been repositioned (moved to the other wrist, adjusted on the ankle, etc.)
- After replacing a node with a new unit
- If directional alerts feel significantly wrong (more than 30° off from expected direction)

---

## Walk-Through Calibration

### What it establishes

The geometric position of each node relative to the belt node (the torso reference). This is necessary for body-frame coordinate transforms — knowing a node's orientation is not sufficient; the system also needs to know where on the body the node is to correctly interpret whether a detection is in front of vs. behind vs. above the wearer.

### Procedure

Walk-through calibration uses the same trilateration principle as AEGIS-MESH's home calibration, adapted for body geometry.

1. After neutral-pose calibration is complete, start the walk-through session:
```bash
curl -X POST http://belt-node.local/api/calibration/walk/start
```

2. Walk your normal walking pace for 20 steps in a straight line. Normal arm swing is expected and helpful. Occasional direction changes are acceptable.

3. The system uses each node's IMU + each node's range sensor (ToF for anklets, radar for wrists) to trilaterate node positions relative to the belt reference.

4. Complete the session:
```bash
curl -X POST http://belt-node.local/api/calibration/walk/complete
curl http://belt-node.local/api/calibration/walk/status
# Returns per-node confidence scores
```

**Confidence thresholds:**
- > 0.85: Excellent calibration. All detection directions should be accurate within ~15°.
- 0.65–0.85: Acceptable. Re-do if alerts feel imprecise.
- < 0.65: Recalibrate. Insufficient node motion diversity during the walk.

### Why walking produces calibration data

During normal walking, each node moves differently:
- Anklet nodes swing with the leg (large position displacement per step)
- Wrist/bracelet nodes swing with arm motion
- Belt node translates with body velocity but minimal rotation

These different motion profiles create a sufficient geometric spread of node-to-belt observations that the trilateration solver can estimate the position of each node in the body frame.

---

## Dynamic Recalibration

After initial calibration, the system continuously refines node position estimates using normal daily movement. Each step of walking contributes IMU and range data that can slightly update the calibration model. This "always-refining" approach means calibration improves with wear time.

The refinement operates within bounds of the initial calibration — it cannot correct a very wrong initial calibration, but it handles drift from node repositioning during the day (a bracelet sliding slightly up the wrist, etc.).

---

## Calibration Failure Modes

**"Node not detected":** The belt controller did not receive BAN transmissions from a node during calibration. Check battery, ensure BAN radio is within range (< 2 m), verify node firmware is running.

**"Low coverage" for a specific node:** The node did not produce enough range measurements during the walk. This can happen if the node's range sensor was blocked during the walk (anklet node blocked by pants fabric, for example). Try again in different clothing or move the node to a slightly different position.

**"Inconsistent IMU":** The node's IMU was in an unusual orientation during calibration (node rotating freely on a loose bracelet). Ensure nodes are secured properly before calibrating.
