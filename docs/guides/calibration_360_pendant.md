# 360° Curved Pendant Calibration Guide

**Project:** SENTINEL-WEAR
**Node Type:** 360 Pendant Node (Variant B — Curved)
**Purpose:** Complete calibration procedures for multi-camera array geometry, stitching, body-frame alignment, and synchronization

---

## 1. Overview

The 360° curved pendant is a signature sensor platform of SENTINEL-WEAR. Its architecture — multiple cameras distributed around a curved PCB arc — requires a comprehensive calibration process to produce clean, geometrically correct 360° panoramas and accurate body-frame localization.

This guide covers all calibration stages:

| Stage | Purpose | When Required | Approximate Duration |
|-------|---------|---------------|----------------------|
| Intrinsic Calibration | Correct individual camera lens distortion | Once per pendant; repeat if lens disturbed | 15-30 minutes |
| Extrinsic Calibration | Establish relative camera positions | Once per pendant; repeat if PCB flexes or camera moved | 10-20 minutes |
| Synchronization Verification | Confirm FSYNC hardware sync | Once per pendant; verify after repair | 5 minutes |
| Stitching Calibration | Define seam positions and blend parameters | Once per pendant; repeat if extrinsic changes | 10-15 minutes |
| Body-Frame Calibration | Position pendant relative to wearer's torso | Each wearing session (or after significant repositioning) | 30-60 seconds |
| Dynamic Calibration | Continuous refinement during operation | Background process; automatic | Automatic |

---

## 2. Architecture Context

### 2.1 Physical Layout

The 360° pendant consists of a curved PCB with cameras distributed at regular angular intervals:

```
                    Pendant Arc (Worn at chest level)
                    
        Camera 0 (0° - Front Center)
            │
            ▼
    ┌───────────────────────────────────────────────────┐
    │                                                   │
    │  Cam 7 (315°)                         Cam 1 (45°) │
    │   ○                                         ○     │
    │                                                   │
    │           Cam 6 (270°)      Cam 2 (90°)          │
    │              ○                    ○              │
    │                                                   │
    │           Cam 5 (225°)      Cam 3 (135°)         │
    │              ○                    ○              │
    │                                                   │
    │  Cam 4 (180° - Rear Center)                       │
    │            ○                                      │
    │                                                   │
    │                 [Vision Processor]                │
    │                 [IMU]                             │
    │                 [CW Radar]                        │
    │                 [Battery]                         │
    │                                                   │
    └───────────────────────────────────────────────────┘
                         │
                    Necklace Chain
```

### 2.2 Camera Configurations

| Configuration | Camera Count | Angular Spacing | Field of View Required | Overlap |
|---------------|--------------|-----------------|------------------------|---------|
| Minimal | 4 | 90° | ≥ 100° | 10° per seam |
| Standard | 6 | 60° | ≥ 70° | 10° per seam |
| Dense | 8 | 45° | ≥ 55° | 10° per seam |
| Ultra | 12 | 30° | ≥ 40° | 10° per seam |

### 2.3 Internal Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         360° PENDANT INTERNAL BUS                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Camera 0 ────┐                                                              │
│  Camera 1 ────┤                                                              │
│  Camera 2 ────┤                                                              │
│  Camera 3 ────┼─── MIPI CSI-2 (Internal Bus) ───► Vision Processor          │
│  Camera 4 ────┤         │                          │                        │
│  Camera 5 ────┤         │                          │                        │
│  Camera 6 ────┤         │                          ├──► Stitching Engine    │
│  Camera 7 ────┘         │                          │                        │
│                          │                          ├──► Encoding (H.264)   │
│                          │                          │                        │
│                          ▼                          └──► UWB/WiFi TX       │
│                    FSYNC Generator                                            │
│                    (Hardware sync pulse to all cameras)                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Stitching Options

| Option | Location | Bandwidth to Belt | Quality | Latency |
|--------|----------|-------------------|---------|---------|
| A — Pendant Stitching | Vision processor on pendant | Single 360° stream (~3-6 Mbps) | Good (2K-2.5K) | Low |
| B — Belt Stitching | Belt node Linux SoM | 8 individual streams (~8-15 Mbps) | High (4K) | Higher |
| C — Progressive | Hybrid | Progressive upgrade | Improves over time | Variable |

---

## 3. Prerequisites

### 3.1 Hardware Required

| Item | Specification | Purpose |
|------|---------------|---------|
| Calibration checkerboard | ArUco 4×4 or ChArUco, A3 or larger | Intrinsic/extrinsic calibration |
| Flat wall space | ≥ 2 m × 2 m, well-lit | Calibration target placement |
| Tripod or stand | For pendant or target | Stable positioning |
| Companion app | Connected to belt node | Running calibration wizard |
| Tape measure | For distance measurements | Camera-to-target distance |
| Spirit level | For ensuring pendant vertical | Body-frame alignment |

### 3.2 Software Required

| Component | Location | Purpose |
|-----------|----------|---------|
| `sentinel-wear-cli` | Host PC or belt node | Calibration commands |
| Companion app | Mobile or desktop | Guided calibration wizard |
| Calibration board PDF | `docs/assets/calibration_board_4x4.pdf` | Print for target |
| `calibration_360_pendant` firmware mode | Pendant MCU | Calibration operation |

### 3.3 Environment Requirements

| Requirement | Specification | Reason |
|-------------|---------------|--------|
| Lighting | Uniform, no harsh shadows | Consistent feature detection |
| Distance | 1-3 meters from pendant to target | Optimal feature resolution |
| Background | Plain, contrasting with target | Reduce false feature detection |
| Wind | Minimal (if outdoors) | Prevent pendant sway |
| Space | Clear 3 m radius around wearer | Full rotation during body-frame calibration |

---

## 4. Intrinsic Camera Calibration

### 4.1 Purpose

Correct lens distortion for each individual camera. Each camera has unique lens characteristics:
- Radial distortion (barrel or pincushion)
- Tangential distortion (lens not parallel to sensor)
- Focal length
- Principal point (optical center)

Without intrinsic calibration:
- Straight lines appear curved
- Distances are inaccurate
- Stitching has geometric errors at seams

### 4.2 Procedure

**Step 1: Print Calibration Board**

```
Print: docs/assets/calibration_board_4x4.pdf
Paper size: A3 minimum, A2 preferred
Square size: 25-30 mm
ArUco dictionary: 4×4, 50 markers
Check: Print at 100% scale, verify square size with ruler
```

**Step 2: Mount Calibration Board**

```
Option A: Mount board on flat wall
- Ensure board is flat and perpendicular to floor
- Good lighting, no glare

Option B: Mount board on rigid backing
- Foam core, rigid cardboard, or thin wood
- Ensure no flex during calibration
```

**Step 3: Enter Calibration Mode**

Via companion app:
```
Settings → Pendant 360° → Calibration → Start Intrinsic Calibration
```

Via CLI:
```bash
sentinel-wear-cli calibrate-intrinsic --node pendant_360 --frames 50
```

**Step 4: Capture Calibration Images**

The pendant captures images from each camera. Move the pendant (or the target) to achieve:

| Required Angle | How to Achieve | Images Needed |
|----------------|-----------------|---------------|
| Front-on (0°) | Target directly in front | 3-5 per camera |
| Left tilt (+30°) | Rotate pendant left | 2-3 per camera |
| Right tilt (-30°) | Rotate pendant right | 2-3 per camera |
| Up tilt (+20°) | Tilt pendant up | 2-3 per camera |
| Down tilt (-20°) | Tilt pendant down | 2-3 per camera |
| Various distances | 1 m, 1.5 m, 2 m, 3 m | 3-5 per distance |

**Total per camera:** 20-30 images minimum, 40-50 recommended

**Step 5: Process Calibration**

Automatic processing occurs on belt node (for Linux SoM variants) or on companion app.

Processing outputs per camera:
```json
{
  "camera_id": 0,
  "focal_length_px": [1425.3, 1428.7],
  "principal_point_px": [956.2, 538.4],
  "distortion_coefficients": [-0.0234, 0.0012, 0.0003, -0.0001, 0.0000],
  "image_size_px": [1920, 1080],
  "reprojection_error_px": 0.34,
  "calibration_quality": "excellent"
}
```

**Quality thresholds:**

| Reprojection Error | Quality | Action |
|--------------------|---------|--------|
| < 0.3 px | Excellent | Proceed |
| 0.3-0.5 px | Good | Proceed |
| 0.5-1.0 px | Acceptable | Consider recapturing challenging angles |
| > 1.0 px | Poor | Recalibrate; check target flatness, lighting |

**Step 6: Verify Calibration**

```bash
# Check calibration status for all cameras
sentinel-wear-cli calibrate-status --node pendant_360 --type intrinsic

# Output:
Camera 0: calibrated, error 0.28 px, timestamp 2024-01-15T14:30:00Z
Camera 1: calibrated, error 0.31 px, timestamp 2024-01-15T14:31:00Z
Camera 2: calibrated, error 0.29 px, timestamp 2024-01-15T14:32:00Z
...
Camera 7: calibrated, error 0.33 px, timestamp 2024-01-15T14:39:00Z
All cameras: CALIBRATED
```

### 4.3 Calibration Data Storage

Intrinsic calibration data is stored:

| Location | Content |
|----------|---------|
| Pendant internal flash | Camera intrinsic parameters (compressed) |
| Belt node SD card | Full calibration backup |
| Companion app cache | Verification copy |

File format: `pendant_360/calibration/intrinsic_camera_N.json`

---

## 5. Extrinsic Camera Calibration

### 5.1 Purpose

Determine the relative position and orientation of each camera in the pendant arc. This establishes:

- Camera center position (x, y, z) in pendant coordinate frame
- Camera orientation (rotation matrix or quaternion)
- Relative geometry between adjacent cameras

Without extrinsic calibration:
- Stitching misalignment
- Seams do not align
- Parallax errors in depth estimation

### 5.2 Procedure

**Prerequisite:** Intrinsic calibration must be complete for all cameras.

**Step 1: Enter Extrinsic Calibration Mode**

```
Companion App: Settings → Pendant 360° → Calibration → Start Extrinsic Calibration
CLI: sentinel-wear-cli calibrate-extrinsic --node pendant_360
```

**Step 2: Position Calibration Target**

Place a checkerboard or ChArUco target such that **at least two adjacent cameras** can see it simultaneously:

```
                    Pendant View
                        
    Camera N      Camera N+1
         \        /
          \      /
           \    /
            \  /
             ▼▼
        ┌───────────┐
        │ Checkerboard│  ← Target positioned to be visible in both cameras
        │   Target    │
        └───────────┘
```

**Step 3: Capture Multi-Camera Views**

Move the target (or walk around with the pendant) to capture:

| View | Target Position | Cameras Covered |
|------|-----------------|-----------------|
| Front | 0° ± 45° | Cameras 0, 1, 7 |
| Left side | 90° ± 45° | Cameras 2, 3 |
| Rear | 180° ± 45° | Cameras 4, 5 |
| Right side | 270° ± 45° | Cameras 6, 7 |

For each view, capture at 3 distances (1.5 m, 2.5 m, 3.5 m) with slight rotations.

**Total views:** 12-20 views covering all camera pairs

**Step 4: Process Extrinsic Calibration**

Processing extracts:
- Checkerboard corners from each camera
- Correspondences between cameras
- Bundle adjustment optimization

Output (simplified):
```json
{
  "calibration_type": "extrinsic",
  "reference_camera": 0,
  "camera_transforms": [
    {
      "camera_id": 0,
      "position_m": [0.0, 0.0, 0.0],
      "rotation_quat": [1.0, 0.0, 0.0, 0.0]
    },
    {
      "camera_id": 1,
      "position_m": [0.042, 0.012, -0.003],
      "rotation_quat": [0.9962, 0.0872, 0.0, 0.0]
    },
    {
      "camera_id": 2,
      "position_m": [0.060, 0.025, -0.005],
      "rotation_quat": [0.9848, 0.1736, 0.0, 0.0]
    }
  ],
  "baseline_avg_m": 0.042,
  "rotation_accuracy_deg": 0.5
}
```

### 5.3 Geometry Validation

After extrinsic calibration, verify the geometry makes sense:

| Check | Expected Range | Failure Indication |
|-------|----------------|-------------------|
| Camera spacing | 40-50 mm between centers | PCB mounting error |
| Angular distribution | 45° ± 2° (for 8-camera) | Camera misalignment |
| Rotation consistency | Smooth progression | Calibration error |
| Baseline consistency | Similar across all pairs | Target detection issues |

**Automated check:**
```bash
sentinel-wear-cli calibrate-verify-geometry --node pendant_360

# Output:
Geometry verification:
  Camera spacing: 44.2 mm (expected: 45 mm) ✓
  Angular distribution: 44.8°, 45.2°, 44.9°, 45.1°, 44.7°, 45.0°, 45.3°, 44.8° ✓
  Rotation consistency: Max deviation 1.2° ✓
  Baseline: 42.8 mm average ✓
RESULT: PASS
```

---

## 6. Synchronization Verification

### 6.1 Purpose

Verify that the FSYNC hardware synchronization signal correctly triggers all cameras simultaneously. Without proper sync:

- Temporal misalignment between cameras
- Stitching artifacts on moving objects
- Incorrect motion estimation

### 6.2 Hardware FSYNC Architecture

```
Vision Processor
       │
       │ (Single GPIO output)
       ▼
    FSYNC Line
       │
       ├─────────┬─────────┬─────────┬─────────┐
       │         │         │         │         │
       ▼         ▼         ▼         ▼         ▼
    Cam 0     Cam 1     Cam 2     ...      Cam 7
    FSYNC     FSYNC     FSYNC             FSYNC
```

**PCB requirements:**
- Single FSYNC trace from vision processor
- Branch to all camera FSYNC inputs
- Trace length matching: target < 10 ns skew
- All traces same length ± 1 mm

### 6.3 Verification Procedure

**Step 1: Enter Sync Test Mode**

```bash
sentinel-wear-cli sync-test --node pendant_360
```

**Step 2: LED Strobe Test**

The pendant's vision processor generates FSYNC pulses while an external high-speed LED strobes at known frequency.

Procedure (automated):
1. Vision processor outputs FSYNC pulse
2. LED strobes (external or built-in test LED)
3. All cameras capture frame
4. Analyze frame timestamps and LED position

**Step 3: Timestamp Analysis**

```json
{
  "test_id": "fsync_20240115_150000",
  "fsync_pulse_timestamp_us": 1705330800000000,
  "camera_frame_timestamps": {
    "camera_0": 1705330800000012,
    "camera_1": 1705330800000011,
    "camera_2": 1705330800000013,
    "camera_3": 1705330800000012,
    "camera_4": 1705330800000014,
    "camera_5": 1705330800000011,
    "camera_6": 1705330800000013,
    "camera_7": 1705330800000012
  },
  "max_skew_us": 3,
  "result": "PASS"
}
```

**Acceptable skew:**

| Skew | Quality | Impact |
|------|---------|--------|
| < 1 µs | Excellent | No visible artifacts |
| 1-5 µs | Good | Minimal artifacts on very fast motion |
| 5-10 µs | Acceptable | Slight artifacts on fast motion |
| > 10 µs | Poor | Visible artifacts; check PCB |

### 6.4 Manual Sync Verification (Alternative)

If no test equipment available:

1. Stand in front of a television or monitor with known refresh rate
2. Run pendant in capture mode
3. Capture frames from all cameras
4. Analyze screen content in frames
5. All frames should show same screen content (or same scan line position for rolling shutter)

---

## 7. Stitching Calibration

### 7.1 Purpose

Define the seam positions where camera images blend together. Stitching calibration creates:

- Seam mask (which pixels belong to which camera)
- Blend weights (smooth transition across seams)
- Projection parameters (equirectangular mapping)

### 7.2 Prerequisites

- Intrinsic calibration complete
- Extrinsic calibration complete
- FSYNC verification passed

### 7.3 Procedure

**Step 1: Enter Stitching Calibration Mode**

```bash
sentinel-wear-cli calibrate-stitching --node pendant_360
```

**Step 2: Capture Reference Environment**

The pendant captures a structured environment with distinctive features:

```
Recommended test scene:
    - Wall with posters/texture
    - Distinct vertical features (doors, windows)
    - Known horizontal lines (shelves, tables)
    - Multiple depth layers (near/far objects)
```

Walk through the environment while the pendant captures reference images from all cameras.

**Step 3: Feature Matching**

Automatic processing:
1. Extract features (ORB, SIFT, or similar) from each camera
2. Match features between adjacent camera pairs
3. Compute homography between camera views
4. Optimize seam positions

**Step 4: Seam Definition**

```json
{
  "seam_config": {
    "projection": "equirectangular",
    "output_resolution": [3840, 1920],
    "seams": [
      {
        "between": [0, 1],
        "azimuth_deg": 22.5,
        "blend_width_px": 50,
        "quality_score": 0.94
      },
      {
        "between": [1, 2],
        "azimuth_deg": 67.5,
        "blend_width_px": 48,
        "quality_score": 0.92
      }
    ]
  }
}
```

**Step 5: Visual Verification**

The companion app displays the stitched panorama with seam overlays:

```
Companion App: Pendant 360° → View → Stitching Preview → Show Seams
```

Verify:
- Straight lines are continuous across seams
- No visible double images
- Smooth color transition
- No gaps or overlaps

### 7.4 Stitching Quality Metrics

| Metric | Target | Measurement |
|--------|--------|--------------|
| Seam error (pixels) | < 3 px | Distance between matched features across seam |
| Color consistency | ΔE < 5 | Color difference across blend region |
| Geometric accuracy | < 1° | Angle error in straight lines |
| Coverage completeness | 100% | No black regions |

---

## 8. Body-Frame Calibration

### 8.1 Purpose

Establish the pendant's position and orientation relative to the wearer's torso. This is required for:

- Body-frame coordinate transforms
- Directional haptic alerts (which node to buzz)
- SLAM body-centric mapping
- Detection direction accuracy

### 8.2 When to Calibrate

| Scenario | Action |
|----------|--------|
| First wearing | Full calibration required |
| Pendant repositioned significantly | Quick recalibration |
| New wearer | Full calibration required |
| After necklace adjustment | Quick recalibration |
| Daily routine | Quick verification (30 seconds) |

### 8.3 Procedure

**Step 1: Enter Body-Frame Calibration Mode**

```
Companion App: Settings → Body-Frame → Calibration → Start Pendant Calibration
```

**Step 2: Neutral Pose (5 seconds)**

Stand in neutral anatomical position:
- Feet hip-width apart
- Arms at sides, relaxed
- Head facing forward
- Torso upright, not slouched

Hold for 5 seconds while the system records:
- Pendant IMU orientation
- Belt IMU orientation
- Relative quaternion between pendant and belt

**Step 3: Motion Sequence (30 seconds)**

Walk at normal pace in a straight line or gentle curve. The system records:
- Pendant motion profile
- Belt motion profile
- Cross-correlation to establish spatial offset

**Step 4: Rotation Sequence (10 seconds)**

Turn left 90°, pause, turn right 90°, pause, turn left 90°, return to facing forward.

The system records:
- Pendant rotation
- Belt rotation
- Verification that rotation is consistent

**Step 5: Calibration Complete**

```json
{
  "calibration_type": "body_frame_pendant",
  "timestamp": "2024-01-15T15:30:00Z",
  "pendant_position_m": [0.0, 0.35, 0.12],   // [right, forward, up] from belt origin
  "pendant_orientation": {
    "quaternion": [0.998, 0.0, 0.063, 0.0],  // Slight forward tilt
    "euler_deg": [0.0, 7.2, 0.0]
  },
  "confidence": 0.94,
  "motion_correlation": 0.97
}
```

### 8.4 Verification

Test the calibration:

```bash
# Have someone walk toward you from the right
# Verify: Right bracelet buzzes (not left)

# Have someone walk toward you from behind-left
# Verify: Left anklet buzzes

# Check companion app: detection direction should match actual approach direction
```

---

## 9. Dynamic Calibration

### 9.1 Purpose

Continuous background refinement during normal operation. Compensates for:

- Small pendant position shifts during activity
- Pendant sway during walking
- Gradual necklace stretch
- Minor calibration drift

### 9.2 Automatic Operation

Dynamic calibration runs continuously in the background:

| Trigger | Action |
|---------|--------|
| Every 100 steps (gait analysis) | Update pendant-to-anklet spatial relationship |
| Every significant rotation | Update pendant orientation relative to belt |
| Every 5 minutes (idle) | Verify static calibration hasn't drifted |

**Configuration:**

```toml
[calibration.dynamic]
enabled = true
update_interval_steps = 100
orientation_update_interval_s = 30
max_deviation_before_alert_deg = 10
max_deviation_before_recal_m = 0.05
```

### 9.3 User Notification

If dynamic calibration detects significant drift:

```
Companion App Alert:
"360° Pendant calibration has drifted. Recommend quick recalibration.
 [Recalibrate Now] [Dismiss]"
```

---

## 10. Calibration File Management

### 10.1 File Locations

| Data | Primary Location | Backup Location |
|------|------------------|------------------|
| Intrinsic parameters | Pendant internal flash | Belt node SD card |
| Extrinsic parameters | Pendant internal flash | Belt node SD card |
| Stitching config | Belt node storage | Companion app cache |
| Body-frame parameters | Belt node storage | Companion app |
| Dynamic calibration logs | Belt node storage | (None) |

### 10.2 File Format

**Intrinsic calibration file** (`intrinsic_camera_N.json`):

```json
{
  "version": "1.0",
  "calibration_date": "2024-01-15T14:30:00Z",
  "camera_id": 0,
  "model": "pinhole",
  "image_size": [1920, 1080],
  "focal_length": [1425.3, 1428.7],
  "principal_point": [956.2, 538.4],
  "distortion": {
    "model": "radtan",
    "coefficients": [-0.0234, 0.0012, 0.0003, -0.0001, 0.0000]
  },
  "reprojection_error_px": 0.28,
  "calibration_tool": "sentinel-wear-cli v1.2.0",
  "images_used": 45
}
```

**Extrinsic calibration file** (`extrinsic.json`):

```json
{
  "version": "1.0",
  "calibration_date": "2024-01-15T15:00:00Z",
  "camera_count": 8,
  "reference_frame": "pendant_center",
  "cameras": [
    {
      "id": 0,
      "position": [0.0, 0.0, 0.0],
      "rotation_matrix": [[1,0,0],[0,1,0],[0,0,1]]
    },
    {
      "id": 1,
      "position": [0.042, 0.012, -0.003],
      "rotation_matrix": [[0.9962,-0.0872,0],[0.0872,0.9962,0],[0,0,1]]
    }
  ],
  "baseline_avg_m": 0.042
}
```

### 10.3 Export and Backup

```bash
# Export all calibration data from pendant
sentinel-wear-cli calibration-export --node pendant_360 --output pendant_360_calibration.zip

# Restore calibration from backup
sentinel-wear-cli calibration-restore --node pendant_360 --input pendant_360_calibration.zip

# Verify calibration integrity
sentinel-wear-cli calibration-verify --node pendant_360
```

---

## 11. Troubleshooting

### 11.1 Intrinsic Calibration Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| High reprojection error (> 1 px) | Target not flat | Use rigid backing |
| High reprojection error | Insufficient coverage | Capture more angles/distances |
| High reprojection error | Motion blur | Stabilize pendant, use shorter exposure |
| Calibration fails | Target not detected | Improve lighting, use larger target |
| Calibration fails | Wrong ArUco dictionary | Verify dictionary matches target |

### 11.2 Extrinsic Calibration Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Camera pairs don't calibrate | Insufficient overlap | Reposition target |
| Inconsistent geometry | PCB flex | Avoid stress on pendant; reseat |
| Large rotation deviations | Wrong camera ordering | Verify camera ID mapping |
| Calibration fails | No shared features | Add more texture to scene |

### 11.3 Synchronization Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Skew > 10 µs | Trace length mismatch | PCB manufacturing issue; may require repair |
| Some cameras out of sync | FSYNC not connected | Check PCB connections |
| Intermittent sync | Loose connection | Inspect connector |

### 11.4 Stitching Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Visible seams | Blend width too narrow | Increase blend_width_px |
| Double images | Extrinsic error | Recalibrate extrinsic |
| Color mismatch | Auto-exposure differences | Enable exposure sync between cameras |
| Gaps in coverage | Insufficient camera FoV | Use wider-angle lenses |

### 11.5 Body-Frame Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Wrong direction alerts | Calibration drift | Quick recalibration |
| Alerts on wrong node | Pendant orientation wrong | Verify pendant hangs correctly |
| SLAM drift | Pendant motion not subtracted | Check body-frame calibration confidence |

---

## 12. Maintenance Schedule

| Task | Frequency | Duration |
|------|-----------|----------|
| Quick body-frame verification | Daily (or each wearing) | 30 seconds |
| Full body-frame calibration | Weekly or after repositioning | 1-2 minutes |
| Intrinsic calibration check | Monthly | 5 minutes (verification) |
| Full intrinsic recalibration | If lens disturbed or after impact | 20-30 minutes |
| Extrinsic recalibration | Rarely needed | 10-15 minutes |
| Synchronization verification | After repair | 5 minutes |
| Stitching verification | Monthly | 2 minutes |

---

## 13. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    360° PENDANT CALIBRATION QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INTRINSIC (Per Camera):                                                     │
│  1. Print ArUco board (A3+)                                                 │
│  2. CLI: calibrate-intrinsic --node pendant_360 --frames 50                 │
│  3. Move pendant/target for 20-50 images per camera                          │
│  4. Target: reprojection error < 0.5 px                                      │
│                                                                              │
│  EXTRINSIC (All Cameras):                                                    │
│  1. Requires intrinsic complete                                             │
│  2. CLI: calibrate-extrinsic --node pendant_360                              │
│  3. Position target for overlapping views                                    │
│  4. 12-20 views covering all camera pairs                                    │
│                                                                              │
│  SYNC VERIFICATION:                                                          │
│  1. CLI: sync-test --node pendant_360                                        │
│  2. Target: max skew < 5 µs                                                  │
│                                                                              │
│  STITCHING:                                                                  │
│  1. CLI: calibrate-stitching --node pendant_360                              │
│  2. Capture reference environment                                            │
│  3. Verify: seam error < 3 px                                                │
│                                                                              │
│  BODY-FRAME (Per Wearing):                                                   │
│  1. Stand in neutral pose (5 s)                                              │
│  2. Walk normally (30 s)                                                     │
│  3. Turn sequence (10 s)                                                     │
│  4. Target: confidence > 0.85                                                 │
│                                                                              │
│  DYNAMIC:                                                                    │
│  - Automatic; enabled by default                                             │
│  - Respond to drift alerts                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Configuration Reference

Complete configuration for 360° pendant calibration:

```toml
[nodes.pendant_360]
enabled = true
variant = "curved"
camera_count = 8

[nodes.pendant_360.calibration]
# Intrinsic calibration settings
intrinsic_calibrated = false
intrinsic_date = ""
intrinsic_reprojection_threshold_px = 0.5
intrinsic_min_frames = 40

# Extrinsic calibration settings
extrinsic_calibrated = false
extrinsic_date = ""
extrinsic_baseline_tolerance_m = 0.005

# Sync verification
fsync_verified = false
fsync_max_skew_us = 5.0

# Stitching settings
stitching_calibrated = false
stitching_projection = "equirectangular"
stitching_output_resolution = [3840, 1920]
stitching_blend_width_px = 50
stitching_seam_error_threshold_px = 3.0

# Body-frame calibration
body_frame_calibrated = false
body_frame_date = ""
body_frame_confidence_threshold = 0.85
body_frame_max_deviation_deg = 10.0

# Dynamic calibration
[calibration.dynamic]
enabled = true
update_interval_steps = 100
orientation_update_interval_s = 30
max_deviation_before_alert_deg = 10
max_deviation_before_recal_m = 0.05

# Stitching mode
[nodes.pendant_360.stitching]
mode = "pendant"                   # "pendant" | "belt" | "progressive"
quality = "standard"               # "standard" | "high"
stream_over_uwb = true
uwb_quality = "2k"                 # "qvga" | "vga" | "2k" | "2.5k"
progressive_enabled = true
progressive_baseline = "qvga"
progressive_target = "720p"
```

---

## 15. Appendix: Calibration Target Specifications

### A.1 ArUco Board (4×4 Dictionary, 50 Markers)

```
Download: docs/assets/calibration_board_4x4.pdf
Paper size: A3 minimum (420 × 297 mm)
Marker size: 25 mm
Border size: 5 mm
Grid: 10 × 7 markers
Dictionary: DICT_4X4_50
```

### A.2 ChArUco Board (Alternative)

```
Download: docs/assets/charuco_board.pdf
Paper size: A2
Checkerboard: 11 × 8 squares
Square size: 30 mm
ArUco markers: 10 × 7 inside checkerboard
Dictionary: DICT_4X4_50
```

### A.3 Verification Pattern

For stitching verification:

```
Download: docs/assets/stitching_verification.pdf
Print on: Large poster paper (A1 or larger)
Pattern: Radial lines from center + grid overlay
Purpose: Verify straight lines across seams
```

---

**End of Document**
