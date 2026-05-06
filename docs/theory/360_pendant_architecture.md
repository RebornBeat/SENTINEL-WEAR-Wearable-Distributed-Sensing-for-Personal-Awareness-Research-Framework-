# 360° Curved Pendant Architecture

**Variant:** Pendant Variant B
**Hardware:** `hardware/schematic/pendant_node/variant_b_360_curved/`

---

## 1. Purpose

The 360° curved pendant is the primary visual sensing platform of SENTINEL-WEAR. By distributing camera modules around the circumference of the pendant arc, it achieves continuous 360° horizontal visual coverage from the chest position — the most geometrically rich sensing position on the human body.

---

## 2. Physical Architecture

The pendant hangs from a necklace chain or torque arc. The PCB follows the pendant's curve — either a fully flexible PCB, a rigid-flex hybrid, or a modular array of rigid PCB segments connected by flexible bridges.

**Camera distribution (8-camera reference configuration):**

| Camera | Position | World Coverage (facing forward) |
|---|---|---|
| 0 | 0° (front center) | Forward hemisphere |
| 1 | 45° (front-right) | Front-right hemisphere |
| 2 | 90° (right) | Right hemisphere |
| 3 | 135° (rear-right) | Rear-right hemisphere |
| 4 | 180° (rear center) | Rear hemisphere |
| 5 | 225° (rear-left) | Rear-left hemisphere |
| 6 | 270° (left) | Left hemisphere |
| 7 | 315° (front-left) | Front-left hemisphere |

With each camera having ≥ 55° FoV, the 45° spacing ensures complete overlap between adjacent cameras at any distance > 0.5 m.

---

## 3. Image Processing Pipeline

### Level A — Belt Node Stitching

Raw frames from all N cameras transmitted to belt node over high-bandwidth link. Belt node Linux SoM stitches to equirectangular.

**Stitching process:**
1. Load per-camera calibration (intrinsics + extrinsic relative to pendant center).
2. For each output pixel in the equirectangular map, compute the spherical coordinate.
3. Project that spherical coordinate into each camera's image plane.
4. Blend contributions from the 1–3 cameras that see that direction (weighted by angular distance from camera center).
5. Output: equirectangular image at configured resolution (default: 3840×1920 for 8 cameras at 2 MP each).

**Computational cost:** Real-time stitching at 3840×1920 × 15 fps ≈ 4–6 W GPU load on Raspberry Pi CM4 VideoCore GPU.

### Level B — Companion App Stitching

Individual compressed camera streams (H.264 per camera) transmitted to companion app. App stitches at full quality using Metal / Vulkan / DirectX.

### Level C — On-Pendant ISP

Dedicated vision processor on pendant (Allwinner V851 or similar) handles stitching locally. Transmits single compressed 360° stream. Reduces belt compute load at the cost of pendant power and complexity.

---

## 4. SLAM Integration

The 360° pendant is the visual SLAM foundation:

**Visual odometry:** Feature extraction from each camera provides dense motion estimation. When the wearer rotates, the pendant rotation is visible as rotation of the visual feature field. The IMU provides prior rotation estimate; visual features refine it.

**Loop closure:** When the pendant camera array revisits a known view (the wearer passes through the same doorway again), the SLAM system recognizes the visual features and corrects accumulated drift. 360° coverage means loop closure is available regardless of which direction the wearer is facing.

**Dense mapping:** Keyframe images from all 8 cameras are used to produce a textured dense mesh of the environment around the wearer. As the wearer moves, new geometry is continuously added.

---

## 5. Synchronization

All N cameras must capture simultaneously for clean stitching. Hardware sync is mandatory:

- A single FSYNC GPIO line connects to all camera FSYNC inputs simultaneously.
- The pendant MCU or Vision MCU drives FSYNC at the configured frame rate.
- All cameras trigger on the same FSYNC rising edge.
- Worst-case timing skew: limited by PCB trace length variation — target < 10 ns between any two cameras.
- Without hardware sync: cameras drift relative to each other at frame rate. Even 1 frame offset (33 ms at 30 fps) causes visible stitching artifacts in fast-motion scenes.

---

## 6. Pendant Sway Compensation

The pendant swings during walking. At 30 fps, a 5° sway between frames creates ~150 pixel shift at 1 m distance (at 3840 pixel equatorial width). Compensation approaches:

**IMU-based warping (software):** The pendant IMU measures angular rate. Between frame captures, angular rate is integrated to estimate sway. The equirectangular output is rotated to cancel the measured sway. Works for < 30°/s sway rates.

**High frame rate capture:** At 60 fps or higher, per-frame sway is smaller. Downstream frame selection picks only frames with sway < 5° from vertical (using IMU). Reduces effective frame rate but improves quality.

**Mechanical damping:** Spring or elastomer isolation between pendant and necklace chain. Reduces sway amplitude by 50–70% for typical walking speeds. Adds 2–5 g to pendant weight.

---

## 7. Companion App — 360° View

The companion app displays the 360° pendant output in three viewing modes:

**Equirectangular (2D):** Standard panoramic view, scrollable left-right. Familiar to users who have used 360° photo applications.

**Spherical (3D interactive):** Renders the equirectangular image on the inside of a sphere. User can look around by swiping. Simulates "being at the pendant position."

**VR headset:** When connected to a VR headset via companion app, provides an immersive first-person view from the pendant's perspective. The user's head rotation maps to viewing direction in the 360° image.

**Threat overlay:** Alert zones from the sparse world model are overlaid on the 360° view as colored regions. An approach alert from the right appears as an orange arc on the right side of the 360° view.
