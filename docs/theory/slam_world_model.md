# SLAM World Model — Dense 3D Reconstruction in SENTINEL-WEAR

**Applies to:** Belt Node Variant B (Linux SoM) and Variant C (High-Performance)
**Primary Sensors:** 360° Curved Pendant (any camera count), Eyewear Node Variant C, Anklet ToF/LiDAR
**Implementation:** `crates/sentinel-slam/`

---

## 1. Overview

SENTINEL-WEAR's world model operates in two simultaneous modes that run in parallel when hardware supports both:

**Mode A — Sparse Probabilistic World Model:**
- **Technology:** OMNI-SENSE → PentaTrack → Body-frame stabilizer
- **Update rate:** 10-50 Hz
- **Latency:** < 20 ms
- **Power:** 100-300 mW
- **Always active:** Yes, on all belt node variants including MCU-only
- **Output:** Tracked entities with position, velocity, prediction centers, classification
- **Use:** All real-time alerting, gait analysis, directional haptic routing

**Mode B — Dense SLAM World Map:**
- **Technology:** LiDAR + cameras → SLAM backend → 3D mesh → Object recognition
- **Update rate:** 0.5-2 Hz (keyframe), continuous background optimization
- **Latency:** 50-200 ms for local updates, minutes for global optimization
- **Power:** 1-3 W depending on compute load
- **Always active:** No, user-configurable; activates on-demand
- **Output:** Full 3D geometry of visited environments, object instances, textured mesh
- **Use:** Forensic review, legal evidence, navigation memory, dense environment reconstruction

**Both modes run simultaneously when enabled.** They are complementary, not mutually exclusive:
- Mode A provides real-time entity tracking overlaid on the dense map
- Mode B provides the geometric context for Mode A's entities
- Object classifications from Mode B improve Mode A's drift profiles
- Mode A's PentaTrack predictions annotate the dense map with motion intent

---

## 2. Architectural Position in the System

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR Software Stack                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────┐         ┌─────────────────────────────────────┐   │
│  │   Mode A — Sparse    │         │      Mode B — Dense SLAM            │   │
│  │                      │         │                                     │   │
│  │  ┌───────────────┐   │         │  ┌─────────────────────────────┐   │   │
│  │  │ OMNI-SENSE    │   │         │  │ Visual Feature Extractor    │   │   │
│  │  │ Sensor Drivers│   │         │  │ (ORB/FAST/SuperPoint)       │   │   │
│  │  └───────┬───────┘   │         │  └─────────────┬───────────────┘   │   │
│  │          │           │         │                │                   │   │
│  │          ▼           │         │                ▼                   │   │
│  │  ┌───────────────┐   │         │  ┌─────────────────────────────┐   │   │
│  │  │ Body-Frame    │   │         │  │ SLAM Backend                │   │   │
│  │  │ Fusion        │   │         │  │ (ORB-SLAM3 / LIO-SAM /      │   │   │
│  │  └───────┬───────┘   │         │  │  OpenVSLAM / Custom)        │   │   │
│  │          │           │         │  └─────────────┬───────────────┘   │   │
│  │          ▼           │         │                │                   │   │
│  │  ┌───────────────┐   │         │                ▼                   │   │
│  │  │ PentaTrack    │   │         │  ┌─────────────────────────────┐   │   │
│  │  │ Bridge        │◄──┼─────────┼──│ Dense World Model           │   │   │
│  │  └───────┬───────┘   │         │  │ (OctoMap / VoxelHash)       │   │   │
│  │          │           │         │  └─────────────┬───────────────┘   │   │
│  │          ▼           │         │                │                   │   │
│  │  ┌───────────────┐   │         │                ▼                   │   │
│  │  │ Sparse Track  │   │         │  ┌─────────────────────────────┐   │   │
│  │  │ Manager       │   │         │  │ Object Recognition          │   │   │
│  │  └───────────────┘   │         │  │ (YOLO / MobileNet)          │   │   │
│  │                      │         │  └─────────────────────────────┘   │   │
│  └─────────────────────┘         └─────────────────────────────────────┘   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      Shared Output Layer                               │  │
│  │                                                                        │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │  │
│  │  │ Tracked Entities│  │ 3D World Mesh   │  │ Object Instance Labels  │ │  │
│  │  │ (sparse)        │  │ (dense)         │  │ (semantic)              │ │  │
│  │  └────────┬────────┘  └────────┬────────┘  └────────────┬────────────┘ │  │
│  │           │                    │                        │             │  │
│  │           └────────────────────┴────────────────────────┘             │  │
│  │                                │                                     │  │
│  │                                ▼                                     │  │
│  │              ┌─────────────────────────────────────┐                  │  │
│  │              │    Unified World Model Interface    │                  │  │
│  │              │    (sentinel-core::WorldModel)      │                  │  │
│  │              └─────────────────┬───────────────────┘                  │  │
│  │                                │                                     │  │
│  │                 ┌──────────────┼──────────────┐                      │  │
│  │                 │              │              │                      │  │
│  │                 ▼              ▼              ▼                      │  │
│  │          ┌───────────┐  ┌───────────┐  ┌─────────────────────┐       │  │
│  │          │ Alerts    │  │ API Server│  │ Companion App       │       │  │
│  │          │ Module    │  │           │  │ (Live View/Export)  │       │  │
│  │          └───────────┘  └───────────┘  └─────────────────────┘       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. SLAM Inputs from Wearable Nodes

### 3.1 360° Curved Pendant (Primary SLAM Anchor)

**Role:** The primary SLAM anchor for SENTINEL-WEAR. Provides the most stable and information-rich visual input.

**Why pendant is the primary anchor:**
- Chest position is the most stable part of the body during locomotion
- Less rotational motion than head (eyewear)
- Centrally located — equal sensitivity in all directions
- 360° coverage eliminates directional blind spots

**Input data:**

| Data Type | Rate | Resolution | Notes |
|-----------|------|------------|-------|
| Camera frames | 15-30 Hz | Configurable per camera | All 4-8 cameras simultaneously via FSYNC |
| Stitched panorama | 15-30 Hz | 1920×960 to 3840×1920 | Depends on compute mode |
| IMU orientation | 200 Hz | Quaternion | Pendant pose in body frame |
| mmWave radar | 10-20 Hz | Point cloud | Complementary depth |
| Acoustic DOA | Event-driven | Direction vector | Event localization |

**Camera synchronization:**
- FSYNC hardware sync ensures all cameras capture simultaneously
- Maximum skew between cameras: < 10 ns
- Critical for clean stitching and accurate depth estimation

**Motion blur mitigation:**
- Pendant IMU measures angular velocity
- Frames with > 5°/s rotation rate flagged as high-motion
- High-motion frames: reduced weight in SLAM optimization
- Can use lower exposure time during fast movement (trades light sensitivity for sharpness)

### 3.2 Eyewear Node (Secondary SLAM Anchor)

**Role:** Provides forward-focused visual odometry and head-pose context.

**Why eyewear is secondary:**
- Head rotation is independent of torso rotation
- Higher motion variability
- Forward-facing coverage only (unless Variant C with side cameras)
- Best for: forward path planning, close-range obstacle detection

**Input data:**

| Data Type | Rate | Resolution | Notes |
|-----------|------|------------|-------|
| Event camera | Event-driven | QVGA+ | Microsecond-latency motion detection |
| Conventional camera | 15-30 Hz | 720p-1080p | Visual odometry, keyframe contribution |
| Side cameras (Variant C) | 15-30 Hz | 720p | Extended forward hemisphere coverage |
| IMU orientation | 200 Hz | Quaternion | Head pose (separate from torso) |

**Head-torso separation:**
- Belt IMU defines torso reference frame
- Eyewear IMU defines head reference frame
- SLAM system maintains both transforms
- Detections from eyewear are correctly attributed to world direction regardless of head position

### 3.3 Anklet Nodes (Ground Reference)

**Role:** Provide ground plane reference and close-range obstacle detection.

**Input data:**

| Data Type | Rate | Range | Notes |
|-----------|------|-------|-------|
| ToF depth | 10-20 Hz | 0.1-6 m | Ground clearance, obstacles |
| Short-range LiDAR | 10-20 Hz | 0.5-12 m | Enhanced geometry |
| IMU orientation | 200 Hz | — | Gait phase, heel-strike |
| Height estimation | Per-step | — | Vertical position relative to floor |

**Contribution to SLAM:**
- Ground plane constraint: Prevents vertical drift
- Floor detection: Anchors Z-axis to physical ground
- Staircase detection: Height change events feed SLAM height adjustment
- Obstacle map: Real-time obstacle positions near feet

### 3.4 Belt Node IMU (Motion Reference)

**Role:** Primary motion reference for all body-frame transformations.

**Input data:**

| Data Type | Rate | Notes |
|-----------|------|-------|
| Linear acceleration | 200 Hz | Body acceleration |
| Angular velocity | 200 Hz | Torso rotation rate |
| Quaternion orientation | 200 Hz | Torso pose estimate |
| Step detection | Event | Gait cycle information |

**Contribution to SLAM:**
- Provides motion prior between camera keyframes
- Enables prediction of camera position during frame intervals
- Critical for visual-inertial odometry (VIO) fusion
- Reduces drift during periods of visual feature scarcity

---

## 4. Mobile SLAM Characteristics

### 4.1 Fundamental Differences from Stationary SLAM

| Aspect | Stationary SLAM (AEGIS-MESH) | Mobile SLAM (SENTINEL-WEAR) |
|--------|------------------------------|------------------------------|
| Sensor position | Fixed in world | Moves with wearer |
| World reference | Static environment | Environment stationary, observer moving |
| Loop closure | Return to same room | Return to same location |
| Map growth | Limited by installation | Grows as wearer moves |
| Drift direction | Accumulates per-node | Accumulates along trajectory |
| Memory management | Static allocation | Dynamic (new areas added, old areas potentially pruned) |

### 4.2 Motion Blur Handling

**The problem:** When the wearer walks, the pendant swings. Camera frames during fast motion are blurry.

**Detection:**
```rust
pub fn is_frame_sharp(imu_angular_velocity: f32, exposure_time_ms: f32) -> bool {
    // Estimate motion blur from angular velocity and exposure time
    let blur_extent_deg = imu_angular_velocity * exposure_time_ms / 1000.0;
    
    // Accept frames with < 1° estimated blur
    blur_extent_deg < 1.0
}
```

**Mitigation strategies:**

| Strategy | Implementation | Tradeoff |
|----------|---------------|----------|
| Adaptive exposure | Reduce exposure during fast motion | Less light, more noise |
| Frame rejection | Skip blurry frames in SLAM optimization | Fewer keyframes |
| IMU-based deblurring | Warp image to compensate for measured motion | Compute cost |
| Multi-rate processing | Low-res always, high-res on sharp frames | Bandwidth |

**Recommended approach:** Adaptive exposure + frame rejection. This provides the best balance of quality and compute efficiency.

### 4.3 Revisitation and Loop Closure

**The advantage:** Wearers naturally revisit locations (hallways, rooms, doorways). Each revisit is a loop closure opportunity.

**Loop closure detection:**
- Bag-of-words visual place recognition (DBoW2 or similar)
- Feature matching between current frame and historical keyframes
- Geometric verification (RANSAC pose estimation)
- Graph optimization when match confirmed

**Processing:**
```
Current frame
    │
    ▼
Feature extraction (ORB/SuperPoint)
    │
    ▼
Place recognition (DBoW2 query)
    │
    ├── No match → Add as new keyframe
    │
    └── Match found
            │
            ▼
        Geometric verification
            │
            ├── Fail → False positive, ignore
            │
            └── Pass → Loop closure detected
                    │
                    ▼
                Pose graph optimization (g2o/GTSAM)
                    │
                    ▼
                Map correction propagates to all affected keyframes
```

**Impact:** Loop closure corrects accumulated drift, improving global map consistency over time.

### 4.4 Map Segmentation

**The need:** Wearers traverse multiple distinct environments (home, office, transit, outdoors). The SLAM system should segment these for efficient storage and selective deletion.

**Segmentation approaches:**

| Approach | Method | Trigger |
|----------|--------|---------|
| Time-based | New segment every N minutes | Simple but imprecise |
| Location clustering | Cluster keyframes by GPS/WiFi fingerprint | More accurate |
| Semantic | Detect "indoor" vs "outdoor" via lighting/features | Context-aware |
| Manual | User creates segment boundary | User control |
| Hybrid | Combine time, location, and semantic | Most robust |

**Segment operations:**
```rust
pub enum MapSegmentOperation {
    Create,                 // Start a new segment
    Merge(SegmentId),       // Merge with existing segment
    Delete(SegmentId),      // Delete segment and all keyframes
    Export(SegmentId),      // Export segment for backup/legal use
    Tag(SegmentId, String), // Add user-defined label
}
```

**Privacy implication:** Users can delete specific segments (e.g., "office visit") without affecting others (e.g., "home map").

### 4.5 Memory Management

**Challenge:** As the wearer moves through the world, the map grows. Memory and storage must be managed.

**Strategies:**

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| Active window | Keep only recent N meters of dense map | Continuous exploration |
| Sparse-dense hybrid | Sparse map everywhere, dense only near current position | Long-duration wear |
| Priority pruning | Remove low-feature areas, keep high-value areas | Feature-rich environments |
| User-controlled | Let user delete old segments | Privacy-focused use |

**Configuration:**
```toml
[slam.memory]
dense_window_radius_m = 10.0     # Keep dense map within this radius
sparse_map_retention_m = 100.0   # Keep sparse map up to this radius
keyframe_max_age_hours = 168     # Prune keyframes older than 1 week (default)
enable_auto_pruning = true
pruning_interval_hours = 24
```

---

## 5. SLAM Backend Options

### 5.1 Algorithm Selection

The SLAM backend is configurable based on available compute and use case:

| Algorithm | Type | Compute | Best For |
|-----------|------|---------|----------|
| ORB-SLAM3 | Visual + Visual-Inertial | Medium | General-purpose, well-tested |
| LIO-SAM | LiDAR-Inertial | Medium-High | LiDAR-focused, good for geometry |
| OpenVSLAM | Visual | Medium | Flexible, supports multiple cameras |
| RTAB-Map | RGB-D / Stereo | High | Rich features, semantic mapping |
| Custom | Optimized for wearable | Variable | Tailored to specific sensors |

### 5.2 ORB-SLAM3 Integration (Default Recommendation)

**Why ORB-SLAM3:**
- Mature, well-tested codebase
- Supports monocular, stereo, and RGB-D cameras
- Visual-inertial mode available
- Good loop closure detection
- Efficient feature extraction (ORB features)

**Wearable adaptations:**
- Multi-camera input: Treat 360° panorama as fisheye camera model
- IMU integration: Use belt IMU as inertial reference
- Dynamic scenes: Filter out moving objects (tracked entities) before SLAM

**Configuration:**
```toml
[slam.backend]
algorithm = "orb_slam3"
mode = "stereo"                 # "mono" | "stereo" | "rgbd" | "fisheye"
enable_visual_inertial = true    # Fuse with IMU
loop_closure = true
relocalization = true

[slam.backend.orb_slam3]
feature_count = 1000             # ORB features per frame
scale_factor = 1.2               # Pyramid scale
level_count = 8                  # Pyramid levels
```

### 5.3 LIO-SAM Integration (LiDAR-Focused)

**Why LIO-SAM:**
- Designed for LiDAR-centric SLAM
- Handles sparse point clouds efficiently
- Good for geometrically complex environments
- Tight LiDAR-IMU integration

**Wearable adaptations:**
- Use anklet ToF and pendant radar as LiDAR sources
- Fuse with camera features for texture
- Useful for users with LiDAR-equipped pendant variants

**Configuration:**
```toml
[slam.backend]
algorithm = "lio_sam"

[slam.backend.lio_sam]
scan_resolution = 0.1            # Voxel size for scan
keyframe_distance = 0.5          # Minimum distance between keyframes (meters)
imu_integrator = "belt_imu"      # Use belt IMU
```

### 5.4 Hybrid Approach

**For maximum capability:** Combine visual and LiDAR SLAM.

```
Visual SLAM (ORB-SLAM3)
    │
    ├── Feature-based loop closure
    ├── Texture mapping
    └── Long-term landmark tracking

LiDAR SLAM (LIO-SAM)
    │
    ├── Accurate local geometry
    ├── Through-occlusion robustness
    └── IMU-integrated odometry

Joint optimization
    │
    └── Fuse both backends via pose graph
```

---

## 6. 360° Camera Processing for SLAM

### 6.1 The Equirectangular Challenge

**Problem:** Standard SLAM algorithms assume planar perspective cameras. A 360° equirectangular image breaks these assumptions.

**Solutions:**

**Option A: Fisheye Camera Model**
- Treat 360° panorama as a fisheye camera
- Use fisheye-capable SLAM (ORB-SLAM3 supports this)
- Camera calibration includes fisheye distortion parameters

**Option B: Cube Map Projection**
- Project equirectangular to 6 planar faces (cube map)
- Run standard perspective SLAM on each face
- Fuse results from all faces

**Option C: Cylindrical Projection**
- Project to cylindrical surface
- Simplifies vertical handling
- Useful for predominantly horizontal motion

**Recommended:** Fisheye model for ORB-SLAM3, as it's a native feature.

### 6.2 Camera Model Configuration

```toml
[slam.camera_model]
type = "fisheye"                # "perspective" | "fisheye" | "equirectangular"

[slam.camera_model.fisheye]
calibration_file = "/etc/sentinel-wear/pendant_360_calibration.yaml"
fov_degrees = 360
rows = 1920
cols = 960
fx = 960.0                      # Focal length x
fy = 960.0                      # Focal length y
cx = 960.0                      # Principal point x
cy = 480.0                      # Principal point y
xi = 1.0                        # Mei model parameter for fisheye
```

### 6.3 Multi-Camera Stitching for SLAM

**When pendant does on-board stitching:**
- Stitching uses calibration parameters from FSYNC-synchronized capture
- Lens distortion corrected per-camera before stitching
- Resulting panorama is geometrically consistent
- SLAM processes the panorama directly

**When belt node does stitching:**
- Raw frames from each camera transmitted to belt
- Belt performs stitching with same calibration parameters
- More compute flexibility on belt
- Higher bandwidth requirement on pendant-to-belt link

---

## 7. Dense World Model Representation

### 7.1 Voxel Grid (OctoMap)

**Representation:** Occupancy probability per voxel in an octree structure.

**Advantages:**
- Efficient storage (octree only stores occupied space)
- Probabilistic occupancy handles sensor noise
- Supports multi-resolution queries

**Configuration:**
```toml
[slam.dense_map]
representation = "octomap"
resolution_m = 0.05             # 5 cm voxels
max_range_m = 10.0              # Maximum sensor range to integrate
clamping_threshold_min = 0.12   # Minimum occupancy probability
clamping_threshold_max = 0.97   # Maximum occupancy probability
```

### 7.2 VoxelHash (For Large Maps)

**Representation:** Spatial hash mapping voxel coordinates to occupancy data.

**Advantages:**
- Better for very large maps (unbounded extent)
- Efficient memory usage for sparse environments
- Fast ray casting for visibility queries

**When to use:** Maps exceeding 100 m traversed distance, or outdoor environments.

### 7.3 Mesh Representation

**Representation:** Triangle mesh surface reconstruction.

**Generation:** Marching cubes on voxel grid, or Poisson reconstruction from point clouds.

**Use cases:**
- Visualization (companion app 3D viewer)
- Object extraction (mesh segmentation)
- Collision detection (for future navigation features)

**Configuration:**
```toml
[slam.mesh]
enabled = true
generation_interval_s = 10.0    # Regenerate mesh every 10 seconds
min_voxels_for_surface = 10     # Minimum voxels to create surface
texture_resolution = 512        # Texture atlas resolution
```

---

## 8. Object Recognition and Semantic Mapping

### 8.1 On-Device Object Detection

**Hardware requirement:** Belt node with NPU (Variant C) or sufficient CPU/GPU for inference.

**Models:**

| Model | Size | Latency | Accuracy | Hardware |
|-------|------|---------|----------|----------|
| YOLOv8-nano | 3 MB | 20-50 ms | Medium | Any Linux SoM |
| YOLOv8-small | 12 MB | 50-100 ms | Good | GPU/NPU preferred |
| MobileNet V3 | 5 MB | 30-80 ms | Medium | Any Linux SoM |
| EfficientDet-lite | 5 MB | 40-80 ms | Good | Any Linux SoM |

**Object categories (default):**

| Category | Examples | Relevance |
|----------|----------|-----------|
| Furniture | chair, table, couch, bed | Navigation context |
| Obstacle | box, bag, debris | Avoidance |
| Person | KnownResident, UnknownPerson | Security context |
| Door | entry_door, interior_door | Segmentation boundaries |
| Vehicle | car, bicycle, motorcycle | Outdoor context |
| Hazard | fire, spill, broken_glass | Safety alerts |

### 8.2 Semantic Map Layers

**The dense map supports multiple semantic layers:**

```
Dense World Model
    │
    ├── Geometry Layer (voxels, mesh)
    │
    ├── Occupancy Layer (probability, free space)
    │
    ├── Texture Layer (RGB color per surface)
    │
    ├── Semantic Layer (object labels per region)
    │
    └── Temporal Layer (when objects were observed)
```

**Query examples:**
```rust
// Get all objects near current position
let nearby_objects = world_model.query_objects()
    .within_radius(current_position, 5.0)
    .of_category("Furniture")
    .execute();

// Get history of a specific region
let region_history = world_model.query_temporal()
    .region(bounding_box)
    .time_range(start_time, end_time)
    .execute();
```

### 8.3 Integration with Sparse Tracking

**Object labels improve PentaTrack tracking:**
- Detected furniture creates "obstacle" regions in PentaTrack's occupancy model
- Person detections from SLAM are fed to PentaTrack as high-confidence tracks
- Door detections create zone boundaries for tracking context

**PentaTrack improves SLAM:**
- Tracked entities are masked out of SLAM feature extraction (prevent dynamic object corruption)
- Velocity estimates from PentaTrack predict camera motion for blur compensation

---

## 9. Power and Thermal Management

### 9.1 Power Consumption by Mode

| Mode | Components Active | Power (W) | Notes |
|------|-------------------|-----------|-------|
| SLAM Idle | Backend loaded, no active processing | 0.5-1.0 | Standby state |
| SLAM Active | Feature extraction, pose estimation | 1.0-2.0 | Standard processing |
| SLAM + Object Recognition | Above + neural inference | 2.0-3.0 | Semantic mapping |
| SLAM + Dense Reconstruction | Above + mesh generation | 2.5-4.0 | High compute |
| SLAM Full | All above + loop closure optimization | 3.0-5.0 | Peak load |

### 9.2 Thermal Throttling

**Problem:** Sustained SLAM processing generates heat. Belt node enclosure temperature must stay below 45°C.

**Thermal management:**
```rust
pub fn thermal_throttle_slam(temperature_c: f32, config: &SlamConfig) -> SlamThrottleAction {
    match temperature_c {
        t if t < 40.0 => SlamThrottleAction::None,
        t if t < 45.0 => SlamThrottleAction::ReduceFrameRate(0.75),  // 75% speed
        t if t < 48.0 => SlamThrottleAction::ReduceFrameRate(0.5),   // 50% speed
        t if t < 50.0 => SlamThrottleAction::DisableObjectRecognition,
        _ => SlamThrottleAction::PauseSlam,
    }
}
```

**Configuration:**
```toml
[slam.thermal]
monitoring_enabled = true
warning_threshold_c = 40.0
throttle_threshold_c = 45.0
pause_threshold_c = 50.0
```

### 9.3 Power-Aware Mode Selection

**Automatic mode selection based on battery:**

```toml
[slam.power_aware]
enabled = true

[[slam.power_aware.rules]]
battery_above = 80
mode = "full"                   # Everything active

[[slam.power_aware.rules]]
battery_above = 50
mode = "standard"               # Disable object recognition

[[slam.power_aware.rules]]
battery_above = 20
mode = "minimal"                # Sparse tracking only

[[slam.power_aware.rules]]
battery_above = 10
mode = "sparse_only"            # Disable SLAM entirely
```

---

## 10. Output: Body-Centric SLAM Map

### 10.1 Map Structure

```rust
pub struct DenseWorldMap {
    /// Unique identifier for this map
    pub id: Uuid,
    
    /// Timestamp of map creation
    pub created_at: DateTime<Utc>,
    
    /// Timestamp of last update
    pub last_updated: DateTime<Utc>,
    
    /// Voxel grid representation
    pub voxels: OctoMap,
    
    /// Surface mesh for visualization
    pub mesh: Option<TriangleMesh>,
    
    /// Object instances detected
    pub objects: Vec<ObjectInstance>,
    
    /// Trajectory of the wearer (body-frame path)
    pub trajectory: Vec<TrajectoryPoint>,
    
    /// Segments (home, office, etc.)
    pub segments: Vec<MapSegment>,
    
    /// Calibration state
    pub calibration: CalibrationState,
    
    /// Loop closure events
    pub loop_closures: Vec<LoopClosureEvent>,
}

pub struct ObjectInstance {
    pub id: Uuid,
    pub category: ObjectCategory,
    pub bounding_box: BoundingBox3D,
    pub position: Point3D,
    pub confidence: f32,
    pub first_observed: DateTime<Utc>,
    pub last_observed: DateTime<Utc>,
    pub observation_count: u32,
}

pub struct MapSegment {
    pub id: Uuid,
    pub label: Option<String>,       // User-defined (e.g., "Home - Kitchen")
    pub bounding_box: BoundingBox3D,
    pub keyframe_ids: Vec<Uuid>,
    pub created_at: DateTime<Utc>,
    pub location_hint: Option<LocationHint>,  // GPS, WiFi fingerprint, etc.
}

pub struct TrajectoryPoint {
    pub timestamp: DateTime<Utc>,
    pub position: Point3D,           // In world frame
    pub orientation: Quaternion,      // Body orientation
    pub velocity: Vector3D,
    pub covariance: Matrix3x3,
}
```

### 10.2 Update Flow

```
Sensor Input (Camera, IMU, ToF)
    │
    ▼
Feature Extraction (ORB features)
    │
    ├── New features → SLAM backend
    │
    └── Tracked features → Pose estimate
            │
            ▼
    Pose Graph Update
            │
            ├── Local: Add to current trajectory
            │
            └── Global: Check for loop closure
                    │
                    ├── No closure → Continue
                    │
                    └── Closure detected → Optimize pose graph
                            │
                            ▼
                    Map Correction
                            │
                            ├── Adjust all affected keyframe poses
                            │
                            └── Update voxel positions
                                    │
                                    ▼
                            World Model Update Event
                                    │
                                    ▼
                        Notify subscribers (API, alerts)
```

### 10.3 Update Rates by Component

| Component | Update Rate | Trigger |
|-----------|-------------|---------|
| Local pose | 10-30 Hz | Every frame |
| Global pose | 0.5-2 Hz | Every keyframe |
| Voxel integration | 10-20 Hz | Every sensor reading |
| Mesh generation | 0.1 Hz | Background process |
| Object recognition | 0.5-2 Hz | Every Nth keyframe |
| Loop closure check | 0.5 Hz | Every keyframe |
| Loop closure optimization | Event-driven | When closure detected |

---

## 11. Use Cases

### 11.1 Security Review

**Scenario:** User wants to review what their environment looked like at a specific time.

**Workflow:**
1. User opens companion app → World View → 3D Map
2. User selects timestamp (e.g., "Yesterday, 3:00 PM")
3. System loads map state at that timestamp from stored keyframes
4. User navigates the 3D environment with entity trajectories overlaid
5. User can see detected objects, tracked entities, and environmental state

**Output:**
- 3D reconstruction of the environment
- Entity positions at selected time
- Object labels and confidence
- 360° video texture for visual review

### 11.2 Evidence Documentation

**Scenario:** User encountered an incident and needs legal evidence.

**Workflow:**
1. User triggers "Export for Evidence" in companion app
2. User selects time range and geographic region
3. System packages:
   - Dense 3D mesh
   - 360° video texture
   - Object instance detections
   - Tracked entity trajectories
   - Timestamp and location data
   - Cryptographic integrity chain
4. Export saved to SD card or transmitted to legal representative

**Integrity chain:**
```json
{
  "export_version": "1.0",
  "map_segment_id": "segment_00123",
  "time_range": {
    "start": "2024-03-15T14:30:00.000Z",
    "end": "2024-03-15T14:35:00.000Z"
  },
  "keyframe_count": 150,
  "object_instances": 12,
  "tracked_entities": 3,
  "file_sha256": "a3f4b2c9...",
  "chain_hash": "b7e8d1f3...",
  "firmware_version": "v1.0.0",
  "export_timestamp": "2024-03-15T15:00:00.000Z"
}
```

### 11.3 Navigation Memory

**Scenario:** User with memory or cognitive challenges wants a record of where they've been.

**Workflow:**
1. SLAM continuously builds map as user moves through environment
2. User can query: "Where was I at 2 PM?"
3. System displays map segment and trajectory for that time
4. User can tag locations: "Doctor's office", "Grocery store"
5. Searchable log of visited locations with timestamps

**Output:**
- Trajectory visualization on map
- Location tags and timestamps
- Duration at each location
- Objects/entities encountered

### 11.4 Research Dataset

**Scenario:** Research institution wants body-frame SLAM data for research.

**Workflow:**
1. User enables "Research Data Collection" mode
2. System records:
   - Raw sensor streams
   - Processed SLAM output
   - Ground truth (if available from user labeling)
3. Data is anonymized (no identifying imagery)
4. Dataset exported in standard format (ROS bag, or custom)
5. Released under appropriate data use agreement

**Dataset contents:**
```
research_dataset/
├── metadata.json
├── trajectory/
│   ├── ground_truth.csv
│   └── estimated.csv
├── sensors/
│   ├── camera/
│   │   ├── cam_0/
│   │   ├── cam_1/
│   │   └── ...
│   ├── imu/
│   │   └── belt_imu.csv
│   ├── lidar/
│   │   └── point_clouds/
│   └── tof/
│       └── depth_frames/
├── slam_output/
│   ├── keyframes/
│   ├── pose_graph.json
│   └── loop_closures.json
└── objects/
    └── detections.json
```

---

## 12. Configuration Reference

```toml
[slam]
enabled = false                   # Enable dense SLAM
mode = "standard"                 # "minimal" | "standard" | "full"

[slam.backend]
algorithm = "orb_slam3"
mode = "fisheye"
enable_visual_inertial = true
loop_closure = true

[slam.dense_map]
representation = "octomap"
resolution_m = 0.05
max_range_m = 10.0

[slam.camera_model]
type = "fisheye"
calibration_file = "/etc/sentinel-wear/pendant_360_calibration.yaml"

[slam.objects]
enabled = true
model = "yolov8_nano"
categories = ["furniture", "person", "door", "vehicle", "hazard"]
inference_interval_frames = 5    # Run detection every 5th frame

[slam.memory]
dense_window_radius_m = 10.0
sparse_map_retention_m = 100.0
keyframe_max_age_hours = 168
enable_auto_pruning = true

[slam.thermal]
monitoring_enabled = true
warning_threshold_c = 40.0
throttle_threshold_c = 45.0
pause_threshold_c = 50.0

[slam.power_aware]
enabled = true

[slam.export]
include_mesh = true
include_textures = true
include_trajectory = true
include_objects = true
integrity_chain = true

[slam.segments]
auto_create = true
segmentation_method = "hybrid"    # "time" | "location" | "semantic" | "hybrid"
min_segment_duration_minutes = 5
```

---

## 13. API Interface

### 13.1 REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/slam/status` | SLAM system status (enabled, mode, processing rate) |
| GET | `/api/slam/map` | Current dense map metadata |
| GET | `/api/slam/map/segments` | List all map segments |
| GET | `/api/slam/map/segments/{id}` | Get specific segment details |
| DELETE | `/api/slam/map/segments/{id}` | Delete a segment |
| GET | `/api/slam/map/mesh` | Download current mesh (OBJ/PLY format) |
| GET | `/api/slam/trajectory` | Get trajectory for time range |
| GET | `/api/slam/objects` | List detected objects in region |
| POST | `/api/slam/export` | Export map segment for evidence |
| POST | `/api/slam/mode` | Change SLAM mode |

### 13.2 WebSocket Streams

| Path | Description |
|------|-------------|
| `/ws/slam/keyframes` | Live keyframe events |
| `/ws/slam/trajectory` | Live trajectory updates |
| `/ws/slam/objects` | Live object detections |
| `/ws/slam/loop_closures` | Loop closure events |

### 13.3 Example Response: Map Status

```json
{
  "enabled": true,
  "mode": "standard",
  "backend": {
    "algorithm": "orb_slam3",
    "state": "tracking",
    "tracking_quality": "good"
  },
  "map": {
    "total_keyframes": 1234,
    "total_voxels": 567890,
    "coverage_m2": 45.2,
    "segments": [
      {
        "id": "segment_00123",
        "label": "Home - Living Room",
        "keyframe_count": 150,
        "area_m2": 18.5
      }
    ]
  },
  "performance": {
    "keyframes_per_second": 0.8,
    "pose_update_hz": 15.2,
    "loop_closures_detected": 12,
    "last_loop_closure": "2024-03-15T14:30:00.000Z"
  },
  "thermal": {
    "temperature_c": 38.5,
    "throttle_state": "none"
  }
}
```

---

## 14. Crate Structure

```
crates/sentinel-slam/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── backend/
│   │   ├── mod.rs
│   │   ├── orb_slam3.rs
│   │   ├── lio_sam.rs
│   │   └── traits.rs              # Common SLAM backend interface
│   ├── camera_model/
│   │   ├── mod.rs
│   │   ├── fisheye.rs
│   │   ├── perspective.rs
│   │   └── equirectangular.rs
│   ├── dense_map/
│   │   ├── mod.rs
│   │   ├── octomap.rs
│   │   ├── voxel_hash.rs
│   │   └── mesh.rs
│   ├── features/
│   │   ├── mod.rs
│   │   ├── orb.rs
│   │   └── superpoint.rs
│   ├── loop_closure/
│   │   ├── mod.rs
│   │   ├── dbow2.rs
│   │   └── geometric_verification.rs
│   ├── objects/
│   │   ├── mod.rs
│   │   ├── detector.rs
│   │   └── categories.rs
│   ├── optimization/
│   │   ├── mod.rs
│   │   ├── pose_graph.rs
│   │   └── bundle_adjustment.rs
│   ├── segmentation/
│   │   ├── mod.rs
│   │   ├── time_based.rs
│   │   ├── location_based.rs
│   │   └── hybrid.rs
│   ├── thermal/
│   │   ├── mod.rs
│   │   └── throttle.rs
│   ├── export/
│   │   ├── mod.rs
│   │   ├── mesh_export.rs
│   │   ├── evidence_package.rs
│   │   └── integrity.rs
│   └── config.rs
└── tests/
    ├── integration_tests.rs
    └── performance_benchmarks.rs
```

---

## 15. Integration with Other Crates

### 15.1 Dependency on sentinel-core

- `WorldModel` type shared across sparse and dense modes
- `DetectionEvent` and `TrackedEntity` types
- Configuration structures

### 15.2 Dependency on sentinel-perception

- Camera frame acquisition
- IMU data processing
- ToF/LiDAR data

### 15.3 Dependency on sentinel-body-frame

- Body-frame coordinate transforms
- IMU fusion for VIO

### 15.4 Dependency on sentinel-tracking

- PentaTrack integration for entity tracking
- Drift profile sharing

### 15.5 Dependency on sentinel-api

- HTTP endpoints for SLAM status
- WebSocket streams for live updates

### 15.6 Dependency on sentinel-storage

- Keyframe storage to SD card
- Map persistence
- Export functionality

---

## 16. Hardware Requirements Summary

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Belt node variant | B (Linux SoM) | C (High-Performance with NPU) |
| RAM | 4 GB | 8 GB |
| Storage | 32 GB SD | 128 GB NVMe |
| Primary sensor | 360° pendant (any camera count) | 360° pendant + eyewear |
| Compute | ARM Cortex-A72 class | ARM Cortex-A76 + NPU |

**MCU-only belt node (Variant A):** Cannot run dense SLAM. Sparse tracking only (Mode A).

---

## 17. Performance Characteristics

| Metric | Minimum | Typical | Maximum |
|--------|---------|---------|---------|
| Local pose latency | 30 ms | 50 ms | 100 ms |
| Keyframe processing | 500 ms | 1-2 s | 5 s |
| Loop closure detection | 100 ms | 500 ms | 2 s |
| Loop closure optimization | 100 ms | 500 ms | 5 s |
| Mesh generation | 1 s | 5-10 s | 30 s |
| Object detection latency | 20 ms | 50-100 ms | 200 ms |
| Map size (per hour) | 10 MB | 50 MB | 200 MB |

---

**End of Document**
