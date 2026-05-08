# Body-Frame Calibration — SENTINEL-WEAR

---

## Overview

SENTINEL-WEAR's body-frame coordinate system requires calibration to produce meaningful directional detections and coordinate transforms. The calibration process establishes:

1. **Orientation reference:** Each node's IMU baseline relative to the torso reference frame
2. **Position estimation:** Each node's geometric position on the body
3. **Camera extrinsics (if applicable):** Camera module positions and orientations for visual coordinate transforms

All calibrations together allow the belt controller to:
- Transform detections from each node's local frame into unified body-frame coordinates
- Stabilize the body frame against wearer motion (translational stabilization)
- Correctly attribute detections regardless of body pose, head orientation, or arm position

---

## Calibration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR Calibration Pipeline                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  Phase 1: Neutral-Pose Calibration                                          │
│  - Establishes IMU reference orientations for all nodes                      │
│  - Required for body-frame stabilization                                    │
│  - Duration: ~5 seconds                                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  Phase 2: Walk-Through Calibration                                          │
│  - Estimates node positions on the body                                     │
│  - Uses motion diversity for geometric trilateration                        │
│  - Duration: ~20 steps                                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Phase 3: Camera Calibration (If Camera-Equipped Nodes Present)             │
│  - Intrinsic calibration (lens distortion)                                  │
│  - Extrinsic calibration (camera position/orientation)                       │
│  - Multi-camera sync verification (360° pendant)                            │
│  - Duration: 5-10 minutes per camera-equipped node                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Phase 4: Ongoing Dynamic Refinement                                        │
│  - Continuous refinement during normal use                                  │
│  - Detects node repositioning                                               │
│  - Improves calibration over time                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: Neutral-Pose Calibration

### What It Establishes

The anatomical alignment of each node's IMU relative to the torso reference frame. Without this calibration:
- The system cannot distinguish "the left bracelet is rotated 45° relative to the torso" from "the torso itself rotated 45°"
- Body-frame fusion would produce incorrect coordinate transforms
- Directional alerts would be misattributed

### Prerequisites

Before starting neutral-pose calibration:

| Prerequisite | Verification |
|--------------|---------------|
| All nodes powered on | Check companion app node list |
| All nodes connected to belt via BAN | All nodes show "Online" status |
| Battery > 20% on all nodes | Low battery affects IMU accuracy |
| Nodes properly secured | Bracelets snug, anklets secure, pendant hanging freely |

### Procedure

**Step 1: Position All Nodes**

Put on all nodes at their intended wearing positions:

| Node | Correct Position | Common Mistakes |
|------|------------------|-----------------|
| Pendant | Center of chest, hanging naturally | Off-center, twisted |
| Belt | Around waist, buckle centered | Too loose, rotated |
| Bracelet Left | Left wrist, face outward | Twisted face-up, too tight |
| Bracelet Right | Right wrist, face outward | Twisted face-up, too tight |
| Anklet Left | Left ankle, above ankle bone | Too low, twisted |
| Anklet Right | Right ankle, above ankle bone | Too low, twisted |
| Eyewear | On face, sitting naturally | Crooked, slipping |

**Step 2: Assume Neutral Anatomical Pose**

Stand in the neutral anatomical pose:

```
Posture Requirements:
├── Feet: Hip-width apart, weight balanced
├── Arms: Hanging naturally at sides (not crossed, not in pockets)
├── Hands: Palms facing body, fingers extended naturally
├── Shoulders: Relaxed, not raised or hunched
├── Head: Facing forward, level (not tilted up/down/left/right)
├── Torso: Upright, not slouched or leaning
├── Hips: Level, not shifted to one side
└── Breathing: Normal (don't hold breath)
```

**Step 3: Initiate Calibration**

Hold the neutral pose for 5 seconds without movement.

Via companion app:
```
Settings → Calibration → Neutral Pose → Start
```

Via API:
```bash
curl -X POST http://belt-node.local/api/calibration/neutral-pose/start
```

**Step 4: Monitor Progress**

The system records IMU data from all nodes for 5 seconds.

```bash
# Check calibration status during recording
curl http://belt-node.local/api/calibration/neutral-pose/status

# Response during recording:
{
  "status": "Collecting",
  "progress": 0.6,
  "nodes_received": ["pendant", "bracelet_left", "bracelet_right", "anklet_left", "anklet_right"],
  "nodes_pending": ["eyewear"]
}

# Response after completion:
{
  "status": "Complete",
  "confidence": 0.97,
  "per_node_confidence": {
    "pendant": 0.98,
    "belt": 0.99,
    "bracelet_left": 0.95,
    "bracelet_right": 0.96,
    "anklet_left": 0.97,
    "anklet_right": 0.98,
    "eyewear": 0.94
  }
}
```

### What the System Records

During neutral-pose calibration, the system records:

**Per-Node Data:**
- IMU accelerometer reading (gravity vector)
- IMU gyroscope bias estimate
- IMU magnetometer reading (if available)
- Current Madgwick/Mahony filter state

**Calibration Outputs:**
- Reference quaternion for each node (orientation at neutral pose)
- Relative quaternion between each node and belt IMU (mounting transform)
- Estimated mounting position (rough, refined in walk-through)

**Body-Frame Definition:**
```
Body Frame (defined by belt IMU at neutral pose):
├── +X: Wearer's right
├── +Y: Wearer's forward
├── +Z: Up (opposite gravity)
└── Origin: Belt IMU position
```

### Quality Assessment

**Confidence Score Interpretation:**

| Score | Interpretation | Action |
|-------|----------------|--------|
| > 0.95 | Excellent | No action needed |
| 0.85 - 0.95 | Good | Acceptable for use |
| 0.70 - 0.85 | Fair | Consider recalibrating |
| < 0.70 | Poor | Recalibrate immediately |

**Common Causes of Low Confidence:**

| Cause | Symptom | Solution |
|-------|---------|----------|
| Movement during calibration | Low per-node confidence | Recalibrate, hold still |
| Node loosely attached | Erratic orientation | Secure node, recalibrate |
| Magnetic interference | Magnetometer conflicts | Move away from metal, recalibrate |
| Low battery | Noisy IMU readings | Charge node, recalibrate |

### When to Redo Neutral-Pose Calibration

Recalibrate neutral pose when:

- A node has been repositioned (moved to other wrist, adjusted on ankle)
- A node is replaced with a new unit
- Directional alerts feel significantly wrong (> 30° off from expected)
- After firmware update that affects IMU processing
- After significant environmental magnetic changes (new appliances, etc.)
- If confidence score drops below 0.70 during normal use

---

## Phase 2: Walk-Through Calibration

### What It Establishes

The geometric position of each node relative to the belt node (torso reference). This is necessary because:
- Knowing a node's orientation is insufficient
- The system needs to know where on the body the node is physically located
- Correct position estimation enables accurate body-frame coordinate transforms

### Principle

Walk-through calibration uses motion diversity for geometric trilateration:

**During walking, different body parts move in distinct patterns:**

| Node | Motion Pattern | Position Information |
|------|----------------|----------------------|
| Belt | Translates forward, minimal rotation | Reference position (origin) |
| Pendant | Translates with torso, slight sway | Position near chest center |
| Bracelet | Swings with arm in pendulum motion | Position at wrist distance from torso |
| Anklet | Alternating forward/backward motion | Position at ankle distance from torso |
| Eyewear | Translates with head, rotation from head motion | Position at head level |

**The solver uses:**
- IMU accelerometer data → motion signature
- IMU gyroscope data → rotation signature
- Range sensor data (ToF/radar) → distance to belt node
- Cross-correlation between nodes → relative position estimation

### Prerequisites

Before walk-through calibration:
- Neutral-pose calibration must be complete (confidence > 0.85)
- Adequate space for 20+ steps in a straight line
- Normal walking surface (flat, no obstacles)
- All nodes reporting online

### Procedure

**Step 1: Start Walk-Through Session**

Via companion app:
```
Settings → Calibration → Walk-Through → Start
```

Via API:
```bash
curl -X POST http://belt-node.local/api/calibration/walk/start
```

**Step 2: Walk Normally**

Walk your normal walking pace for 20-30 steps in a straight line.

```
Walking Guidelines:
├── Pace: Normal walking speed (not fast, not slow)
├── Arm swing: Natural arm swing (helps bracelet calibration)
├── Direction: Straight line (don't turn during recording)
├── Steps: 20-30 steps minimum
└── Environment: Clear path, no obstacles
```

**What the system observes:**
- Each anklet alternates between forward and backward motion relative to torso
- Each bracelet swings in an arc
- The pendant translates with torso with minimal swing
- Eyewear translates and rotates with head motion
- Range sensors measure distance variations between nodes

**Step 3: Complete Session**

After 20-30 steps:

```bash
curl -X POST http://belt-node.local/api/calibration/walk/complete
```

**Step 4: Review Results**

```bash
curl http://belt-node.local/api/calibration/walk/status

# Response:
{
  "status": "Complete",
  "overall_confidence": 0.91,
  "per_node": {
    "pendant": {
      "position_m": [0.0, 0.15, 0.35],
      "confidence": 0.93,
      "position_label": "chest_center"
    },
    "bracelet_left": {
      "position_m": [-0.25, 0.05, 0.85],
      "confidence": 0.88,
      "position_label": "left_wrist"
    },
    "bracelet_right": {
      "position_m": [0.25, 0.05, 0.85],
      "confidence": 0.90,
      "position_label": "right_wrist"
    },
    "anklet_left": {
      "position_m": [-0.08, -0.05, 0.05],
      "confidence": 0.92,
      "position_label": "left_ankle"
    },
    "anklet_right": {
      "position_m": [0.08, -0.05, 0.05],
      "confidence": 0.91,
      "position_label": "right_ankle"
    },
    "eyewear": {
      "position_m": [0.0, 0.08, 1.55],
      "confidence": 0.86,
      "position_label": "head"
    }
  },
  "recommendations": []
}
```

### Quality Assessment

**Confidence Interpretation:**

| Score | Quality | Expected Accuracy |
|-------|---------|-------------------|
| > 0.85 | Excellent | Direction accurate within ~15° |
| 0.70 - 0.85 | Good | Direction accurate within ~25° |
| 0.55 - 0.70 | Fair | Direction accurate within ~40° |
| < 0.55 | Poor | Recalibrate |

**Why Walking Works:**

The key insight is that different body parts move in distinct patterns during walking:

```
Anklet Motion During Walking:
Step 1 (Left foot forward):
  - Left anklet: 0.6m forward of belt (toe-off)
  - Right anklet: 0.2m behind belt (stance)
Step 2 (Right foot forward):
  - Left anklet: 0.2m behind belt (stance)
  - Right anklet: 0.6m forward of belt (toe-off)

Pattern: Alternating forward/backward displacement
→ Reveals: Ankle is at foot level, ~0.6m stride length
→ Position: ~0.8m below belt, alternating ±0.15m lateral
```

### Troubleshooting Low Confidence

**Symptom: Low confidence on specific node**

| Possible Cause | Diagnostic | Solution |
|----------------|------------|----------|
| Range sensor blocked | Check node logs for range readings | Remove obstruction, recalibrate |
| Node loose/flopping | Visual inspection | Secure node, recalibrate |
| Insufficient motion | Calibration log shows low variance | Walk faster/longer, recalibrate |
| BAN packet loss | High retry count in logs | Check antenna position, recalibrate |

**Symptom: All nodes low confidence**

| Possible Cause | Diagnostic | Solution |
|----------------|------------|----------|
| Belt not moving naturally | Calibration shows zero torso motion | Walk naturally, not robotically |
| BAN interference | High latency in logs | Enable antenna diversity, recalibrate |
| Calibration environment | Metal floors/walls | Move to different location, recalibrate |

---

## Phase 3: Camera Calibration

For nodes equipped with cameras (pendant with camera, 360° pendant, bracelet with camera, eyewear with camera), additional calibration is required for visual coordinate transforms.

### 3.1 Camera Intrinsic Calibration

**Purpose:** Correct lens distortion and establish camera internal parameters.

**Required equipment:**
- Printed calibration target (ArUco markers or checkerboard)
- Minimum 40 images from different angles/distances

**Procedure:**

1. Print calibration target:
```bash
sentinel-wear-cli print-calibration-target --type aruco_4x4 --size a4
```

2. Start intrinsic calibration:
```bash
curl -X POST http://belt-node.local/api/calibration/camera/intrinsic/start \
  -d '{"node_id": "pendant", "target_type": "aruco_4x4"}'
```

3. Capture images (via companion app or automated):
- Hold target at different distances (0.5m, 1m, 2m)
- Rotate target to different angles
- Move target to different positions in camera FOV
- Minimum 40 images, recommended 60-100

4. Process calibration:
```bash
curl -X POST http://belt-node.local/api/calibration/camera/intrinsic/process \
  -d '{"node_id": "pendant"}'
```

5. Review results:
```bash
curl http://belt-node.local/api/calibration/camera/intrinsic/result?node_id=pendant

# Response:
{
  "node_id": "pendant",
  "rms_reprojection_error_px": 0.42,
  "image_count": 52,
  "camera_matrix": [...],
  "distortion_coefficients": [...],
  "image_size_px": [1920, 1080],
  "quality": "excellent"
}
```

**RMS Reprojection Error Thresholds:**

| RMS Error | Quality | Action |
|-----------|---------|--------|
| < 0.5 px | Excellent | Accept calibration |
| 0.5 - 1.0 px | Good | Accept calibration |
| 1.0 - 2.0 px | Fair | Consider recapturing |
| > 2.0 px | Poor | Recalibrate |

### 3.2 Camera Extrinsic Calibration

**Purpose:** Establish camera position and orientation relative to the node's IMU and body frame.

**Procedure:**

1. Complete intrinsic calibration first (required)
2. Place calibration target at known position relative to body
3. Capture calibration images with target visible
4. Process to get extrinsic transform

```bash
curl -X POST http://belt-node.local/api/calibration/camera/extrinsic/start \
  -d '{"node_id": "pendant"}'

# Capture images...

curl -X POST http://belt-node.local/api/calibration/camera/extrinsic/process \
  -d '{"node_id": "pendant"}'
```

**Output:**
```json
{
  "node_id": "pendant",
  "camera_position_m": [0.02, 0.08, 0.04],
  "camera_rotation_quat": [w, x, y, z],
  "imu_to_camera_transform": [...],
  "confidence": 0.91
}
```

### 3.3 360° Pendant Camera Calibration

The 360° curved pendant requires additional calibration due to multiple cameras.

**Step 1: Individual Camera Intrinsics**

Calibrate each camera independently:

```bash
# Calibrate all 8 cameras sequentially
for i in {0..7}; do
  curl -X POST http://belt-node.local/api/calibration/camera/intrinsic/start \
    -d '{"node_id": "pendant_360", "camera_index": $i, "target_type": "aruco_4x4"}'
  
  # Capture images for this camera...
  
  curl -X POST http://belt-node.local/api/calibration/camera/intrinsic/process \
    -d '{"node_id": "pendant_360", "camera_index": $i}'
done
```

**Step 2: Inter-Camera Extrinsic Calibration**

Establish relative positions between cameras:

```bash
curl -X POST http://belt-node.local/api/calibration/camera/360/extrinsic/start \
  -d '{"node_id": "pendant_360"}'
```

**Procedure:**
1. Print large calibration target (A3 or larger)
2. Position target visible to multiple cameras simultaneously
3. Move target to capture overlapping views
4. System computes relative transforms between cameras

**Step 3: FSYNC Synchronization Verification**

Verify all cameras trigger simultaneously:

```bash
curl -X POST http://belt-node.local/api/calibration/camera/360/fsync-test \
  -d '{"node_id": "pendant_360", "samples": 100}'

# Response:
{
  "node_id": "pendant_360",
  "sync_quality": "excellent",
  "max_skew_ns": 8,
  "mean_skew_ns": 3,
  "failed_cameras": []
}
```

**FSYNC Quality Thresholds:**

| Max Skew | Quality | Action |
|----------|---------|--------|
| < 10 ns | Excellent | Accept |
| 10 - 50 ns | Good | Accept |
| 50 - 100 ns | Fair | Check cabling |
| > 100 ns | Poor | Check FSYNC line |

**Step 4: Stitching Calibration**

Calibrate the stitching parameters:

```bash
curl -X POST http://belt-node.local/api/calibration/camera/360/stitching/start \
  -d '{"node_id": "pendant_360"}'
```

**Procedure:**
1. Place target in room center
2. Rotate pendant to capture target from all cameras
3. System computes overlap regions and seam placement

**Step 5: Verify Stitching Quality**

```bash
curl http://belt-node.local/api/calibration/camera/360/stitching/result?node_id=pendant_360

# Response:
{
  "node_id": "pendant_360",
  "seam_max_error_px": 2.1,
  "overlap_consistency": 0.97,
  "resolution": [3840, 1920],
  "status": "excellent"
}
```

**Visual Verification:**

In the companion app, view the live 360° panorama:
- Check for visible seams at camera boundaries
- Verify horizon line is continuous
- Check for distortion spikes at any angular position
- If seams are visible, re-run stitching calibration

### 3.4 Camera Calibration Frequency

| Situation | Recommended Action |
|-----------|-------------------|
| First deployment | Full calibration (intrinsic + extrinsic) |
| After camera replacement | Full calibration |
| After significant drop/impact | Verify extrinsic, re-run if needed |
| Every 6 months | Verify intrinsic, re-run if RMS > 1.0 px |
| After firmware update affecting camera | Verify all parameters |

---

## Phase 4: Ongoing Dynamic Refinement

### Continuous Refinement

After initial calibration, the system continuously refines estimates during normal use:

**Data sources for refinement:**
- IMU motion patterns during daily activity
- Range sensor measurements
- Visual feature matching (camera-equipped nodes)
- Cross-node observation correlation

**Refinement characteristics:**
- Operates within bounds of initial calibration
- Cannot correct severely wrong initial calibration
- Handles gradual drift from node repositioning
- Improves accuracy over time with more data

**Configuration:**

```toml
[calibration.dynamic_refinement]
enabled = true
update_rate_hz = 1                    # Refinement update frequency
max_position_drift_m = 0.05           # Maximum allowed position drift before alert
max_orientation_drift_deg = 5         # Maximum allowed orientation drift
confidence_threshold = 0.85           # Minimum confidence for auto-update
learning_rate = 0.01                 # How quickly to incorporate new data
```

### Drift Detection

The system monitors for calibration drift:

```bash
curl http://belt-node.local/api/calibration/drift-status

# Response:
{
  "drift_detected": false,
  "per_node": {
    "pendant": {"position_drift_m": 0.008, "orientation_drift_deg": 1.2},
    "bracelet_left": {"position_drift_m": 0.012, "orientation_drift_deg": 2.1},
    "bracelet_right": {"position_drift_m": 0.009, "orientation_drift_deg": 1.8},
    "anklet_left": {"position_drift_m": 0.015, "orientation_drift_deg": 2.5},
    "anklet_right": {"position_drift_m": 0.014, "orientation_drift_deg": 2.3}
  }
}
```

**When drift exceeds thresholds, the system:**
1. Logs a calibration warning
2. Notifies via companion app
3. Continues operation with degraded confidence
4. Recommends recalibration

---

## Calibration for Extreme Velocity Detection

Extreme velocity detection (projectile detection) places specific demands on calibration:

### Timing Synchronization Requirements

| Requirement | Target | Calibration Impact |
|-------------|--------|-------------------|
| Inter-node time sync | < 1 ms | UWB time sync calibration |
| IMU timestamp alignment | < 100 µs | Part of neutral-pose calibration |
| Event camera timing | < 10 µs | Camera calibration includes timing |

### Position Accuracy Requirements

For accurate threat trajectory estimation:

| Node | Position Accuracy Needed | Calibration Method |
|------|------------------------|-------------------|
| Pendant | ± 2 cm | Full walk-through + camera extrinsic |
| Eyewear | ± 2 cm | Full walk-through + camera extrinsic |
| Bracelets | ± 3 cm | Full walk-through |
| Anklets | ± 3 cm | Full walk-through |

### Extreme Velocity Calibration Procedure

Additional steps for extreme velocity capability:

```bash
# Enable extreme velocity calibration mode
curl -X POST http://belt-node.local/api/calibration/extreme-velocity/start

# System performs additional timing and position refinements
# Duration: 2-3 minutes of varied motion

# Complete
curl -X POST http://belt-node.local/api/calibration/extreme-velocity/complete
```

**Verification:**
```bash
curl http://belt-node.local/api/calibration/extreme-velocity/status

# Response:
{
  "ready": true,
  "timing_sync_quality_ns": 850,
  "position_accuracy_m": 0.018,
  "coverage_degrees": 360,
  "extreme_velocity_capable": true
}
```

---

## Calibration File Storage

Calibration data is stored on the belt node:

**Storage location:** `/var/lib/sentinel-wear/calibration/`

**Files:**
```
calibration/
├── neutral_pose.json           # Neutral-pose calibration data
├── walkthrough.json            # Walk-through position estimates
├── camera/
│   ├── pendant_intrinsic.json
│   ├── pendant_extrinsic.json
│   ├── pendant_360_intrinsic.json
│   ├── pendant_360_extrinsic.json
│   └── pendant_360_stitching.json
├── extreme_velocity.json       # Extreme velocity calibration
└── calibration_history.json   # History for drift analysis
```

**Backup:**
```bash
# Export calibration data
curl http://belt-node.local/api/calibration/export > calibration_backup.json

# Restore calibration data
curl -X POST http://belt-node.local/api/calibration/import \
  --data-binary @calibration_backup.json
```

---

## Troubleshooting Guide

### Symptom: "Node not detected" during calibration

**Diagnosis:**
- BAN communication failure
- Node powered off or battery depleted
- Node outside BAN range

**Resolution:**
```bash
# Check node connectivity
curl http://belt-node.local/api/nodes

# If node shows offline:
# 1. Check node battery
# 2. Verify node is within 2 meters of belt
# 3. Check for BAN interference (disable WiFi 2.4 GHz temporarily)
# 4. Re-pair node if necessary
```

### Symptom: "Low coverage" for specific node

**Diagnosis:**
- Range sensor blocked during walk-through
- Node motion too constrained
- Clothing obstructing sensor

**Resolution:**
1. Check for sensor obstructions (tight pants over anklet, long sleeves over bracelet)
2. Ensure natural arm swing during walk-through
3. Walk longer distance (30+ steps)
4. Try different walking speed

### Symptom: "Inconsistent IMU" warning

**Diagnosis:**
- Node loose/flopping on body
- IMU malfunction
- Magnetic interference

**Resolution:**
1. Verify node is securely attached
2. Move away from strong magnetic sources
3. If problem persists, check IMU health:
```bash
curl http://belt-node.local/api/nodes/{node_id}/imu-health
```

### Symptom: "Calibration drift detected" alert

**Diagnosis:**
- Node has been repositioned
- Gradual loosening of strap/band
- Environmental changes

**Resolution:**
1. Verify node position visually
2. If repositioned, recalibrate neutral-pose and walk-through
3. If position correct but drift continues, check for:
   - Strap/band loosening
   - Battery degradation affecting IMU
   - Environmental magnetic changes

### Symptom: 360° stitching visible seams

**Diagnosis:**
- Incomplete extrinsic calibration
- Camera physical displacement (drop/impact)
- FSYNC timing issue

**Resolution:**
1. Re-run inter-camera extrinsic calibration
2. Verify FSYNC quality:
```bash
curl -X POST http://belt-node.local/api/calibration/camera/360/fsync-test \
  -d '{"node_id": "pendant_360", "samples": 100}'
```
3. If cameras physically displaced, re-run intrinsic calibration
4. Visual inspection of 360° pendant for damage

### Symptom: Directional alerts consistently wrong

**Diagnosis:**
- Calibration confidence degraded
- Reference frame mismatch
- IMU bias drift

**Resolution:**
```bash
# Check current calibration confidence
curl http://belt-node.local/api/calibration/confidence

# If confidence < 0.70, recalibrate

# If confidence good but direction wrong:
# Check body-frame alignment
curl http://belt-node.local/api/calibration/body-frame-verify

# Response shows current alignment vs expected
```

---

## Calibration API Reference

### Neutral-Pose Calibration

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/calibration/neutral-pose/start` | Begin neutral-pose calibration |
| GET | `/api/calibration/neutral-pose/status` | Get calibration progress/status |
| POST | `/api/calibration/neutral-pose/cancel` | Cancel ongoing calibration |

### Walk-Through Calibration

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/calibration/walk/start` | Begin walk-through calibration |
| GET | `/api/calibration/walk/status` | Get calibration results |
| POST | `/api/calibration/walk/complete` | Manually complete calibration |
| POST | `/api/calibration/walk/cancel` | Cancel ongoing calibration |

### Camera Calibration

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/calibration/camera/intrinsic/start` | Begin intrinsic calibration |
| POST | `/api/calibration/camera/intrinsic/process` | Process captured images |
| GET | `/api/calibration/camera/intrinsic/result` | Get intrinsic calibration results |
| POST | `/api/calibration/camera/extrinsic/start` | Begin extrinsic calibration |
| POST | `/api/calibration/camera/extrinsic/process` | Process extrinsic calibration |
| GET | `/api/calibration/camera/extrinsic/result` | Get extrinsic results |

### 360° Camera Calibration

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/calibration/camera/360/extrinsic/start` | Begin inter-camera calibration |
| POST | `/api/calibration/camera/360/fsync-test` | Test camera synchronization |
| POST | `/api/calibration/camera/360/stitching/start` | Begin stitching calibration |
| GET | `/api/calibration/camera/360/stitching/result` | Get stitching results |

### General Calibration

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/calibration/confidence` | Get overall calibration quality |
| GET | `/api/calibration/drift-status` | Check for calibration drift |
| GET | `/api/calibration/export` | Export calibration data |
| POST | `/api/calibration/import` | Import calibration data |
| POST | `/api/calibration/extreme-velocity/start` | Begin extreme velocity calibration |
| GET | `/api/calibration/extreme-velocity/status` | Get extreme velocity calibration status |

---

## Calibration Best Practices

### Timing

| Situation | Recommendation |
|-----------|----------------|
| First setup | Full calibration in quiet environment |
| After firmware update | Verify calibration, re-run if needed |
| Before critical use | Verify calibration confidence |
| Regular maintenance | Weekly confidence check |

### Environment

| Factor | Recommendation |
|--------|----------------|
| Magnetic interference | Avoid large metal objects during calibration |
| RF interference | Disable WiFi 2.4 GHz during BAN-sensitive calibration |
| Space | Adequate room for 30+ step walk-through |
| Lighting | Good lighting for camera calibration |

### Node Placement

| Node | Key Consideration |
|------|------------------|
| Bracelets | Consistent orientation (face outward) |
| Anklets | Secure above ankle bone |
| Pendant | Hanging freely, not twisted |
| Belt | Centered on waist |
| Eyewear | Sitting naturally on face |

---

## Summary

SENTINEL-WEAR's calibration establishes the body-frame coordinate system essential for accurate directional detection and tracking:

| Phase | Purpose | Duration | Accuracy Target |
|-------|---------|----------|-----------------|
| Neutral-Pose | IMU reference orientations | 5 seconds | ±2° orientation |
| Walk-Through | Node position estimation | 20-30 steps | ±3 cm position |
| Camera | Visual coordinate transforms | 5-10 min/node | < 1 px reprojection |
| Dynamic | Continuous refinement | Ongoing | Maintains accuracy |

**Key insights:**
- Calibration quality directly affects detection accuracy
- Multi-modal calibration (IMU + range + visual) provides robust estimates
- Continuous refinement handles daily drift
- Extreme velocity detection requires additional timing calibration
- 360° pendant requires dedicated multi-camera calibration

**For best results:**
1. Calibrate in a consistent environment
2. Ensure nodes are properly secured before calibration
3. Verify calibration confidence before critical use
4. Monitor drift indicators and recalibrate when needed
5. Back up calibration data after successful calibration
