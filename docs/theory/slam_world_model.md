# SLAM World Model — Dense 3D Reconstruction in SENTINEL-WEAR

**Applies to:** Belt Node Variant B (Linux SoM), with pendant 360° cameras and/or eyewear Variant C
**Implementation:** `crates/sentinel-slam/`

---

## 1. Overview

SENTINEL-WEAR's world model operates in two simultaneous modes:

**Mode A — Sparse:** PentaTrack-based body-centric probabilistic tracking. Always active. Low power. Fast (< 20 ms update).

**Mode B — Dense SLAM:** Full 3D reconstruction of the wearer's environment as they move. Requires Linux SoM belt node and 360° pendant cameras or eyewear Variant C visual array.

---

## 2. SLAM Inputs from Wearable Nodes

**360° Curved Pendant (Variant B):**
- 8 camera streams at 45° intervals = no blind spots at chest level
- IMU provides pendant pose for sensor-to-world transform
- This is the primary SLAM anchor: chest level, omnidirectional coverage, stable relative to torso

**Eyewear Node (Variant C — full array):**
- Forward + side cameras = 180°+ visual coverage from head level
- Head IMU provides head pose (separate from torso)
- Contributes forward visual odometry, loop closure on revisited areas
- Wide baseline between forward and side cameras enables depth from stereo

**Anklet ToF:**
- Ground plane height and obstacle detection
- Constrains vertical drift in SLAM (floor-level reference)

**Belt IMU:**
- Primary motion reference
- Provides velocity estimate for map prediction between sensor updates

---

## 3. Mobile SLAM Characteristics

Unlike room-mounted SLAM (AEGIS-MESH), SENTINEL-WEAR SLAM is **mobile** — the sensors move with the wearer. This creates unique challenges:

**Motion blur:** When the wearer walks, the pendant swings. Camera frames during high-motion moments are rejected by the SLAM system (too blurry for reliable feature extraction). Only frames with acceptable motion score are used as keyframes.

**Revisitation:** The wearer returns to the same locations (doorways, kitchen, living room). Each revisit provides loop closure opportunity that corrects accumulated drift.

**Privacy boundary:** The SLAM map is stored locally on the belt node SD card. It is not transmitted anywhere unless the user configures cloud backup or legal export.

**Map segmentation:** The SLAM system segments the map by location cluster (home, office, transit, outdoor). Users can selectively delete specific location segments.

---

## 4. Output: Body-Centric SLAM Map

The dense world model updates as the wearer moves:

**Immediate update (< 100 ms):** New LiDAR/ToF readings integrate into the occupancy grid around the wearer's current position.

**Keyframe update (every 0.5–2 s):** Camera keyframe processed for feature matching, map extension, and loop closure.

**Global consistency (background, minutes):** Loop closure optimization runs in the background. When the wearer revisits a known area, the map is adjusted for global consistency.

**Companion app view:** Users see the body-centric 3D map update in real-time. As they walk through their environment, the 3D model builds around them. On the companion app: **World View → 3D Map → Current Environment**.

---

## 5. Use Cases

**Security review:** "What did my environment look like at 3pm yesterday?" — scrub the 3D map timeline to replay the state of the environment at any recorded moment.

**Evidence documentation:** Encountered an incident? Export the 3D map segment with timestamps and integrity manifest. The map includes 360° video texture for every area the wearer passed through.

**Navigation memory:** For users with memory or cognitive challenges, the SLAM map provides a complete visual record of everywhere visited, with timestamps and event annotations.

**Research:** The body-frame SLAM dataset (position + sensor data from a mobile human) is a research contribution. Phase 5 of the roadmap includes a public open-data release of anonymized body-frame SLAM trajectories.
