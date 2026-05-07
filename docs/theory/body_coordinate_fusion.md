# Body-Coordinate Fusion — Architecture and Drift Correction

**Project:** SENTINEL-WEAR
**Domain:** Multi-node sensor fusion in non-rigid coordinate frames
**Implementation:** `crates/sentinel-body-frame/`
**Dependencies:** OMNI-SENSE IMU fusion, PentaTrack tracking framework

---

## 1. The Problem

A wearable sensor mesh observes the world from multiple body-worn positions — pendant, bracelets, belt, anklets, optionally eyewear. Each position has its own local frame defined by the orientation of the body part it's attached to. The body itself is a non-rigid multi-segment frame: the head rotates relative to the torso, arms move relative to the torso, legs move relative to the torso, and the torso rotates and translates through the world.

A naive fusion that simply combines each node's local-frame detections produces a perception field that rotates when the wearer turns, lurches when the wearer walks, and reports phantom approaches whenever the wearer gestures. The fusion problem is to subtract wearer motion and produce a stable body-frame perception field — one in which "approach from the right rear" means an actual approach from the right rear, regardless of which way the wearer is facing.

This document specifies the architecture, calibration procedures, drift correction mechanisms, and integration with PentaTrack for predictive tracking.

---

## 2. Coordinate Frames

### 2.1 Frame Definitions

Three frames matter for body-coordinate fusion:

| Frame | Symbol | Origin | Orientation |
|-------|--------|--------|-------------|
| **World frame** | W | Fixed external reference | Geographic or magnetic north |
| **Torso frame** | T | Belt node position (waist/sternum) | Anatomical axes of torso |
| **Node frame** | Nᵢ | Node *i*'s physical position | Local IMU orientation |

### 2.2 Torso Frame Definition

The torso frame (T) is SENTINEL-WEAR's primary perception frame:

```
Torso Frame (T) — Belt IMU Reference
├── Origin: Belt node center
├── +X axis: Wearer's right (anatomical right)
├── +Y axis: Wearer's forward (direction torso is facing)
└── +Z axis: Up (opposite gravity vector)
```

**All detections from all nodes are transformed into this frame before fusion.**

### 2.3 Transform Notation

| Transform | Notation | Meaning |
|-----------|----------|---------|
| Node-to-Torso | ᵀTNᵢ | Rotation from node *i*'s frame to torso frame |
| Torso-to-World | ᵂTT | Rotation from torso frame to world frame |
| Node-to-World | ᵂTNᵢ | Combined: ᵂTT × ᵀTNᵢ |

---

## 3. Per-Node Transformation

### 3.1 Observation Model

Each node observes the world in its own local frame. A radar return at node *i* reports:

| Measurement | Symbol | Description |
|-------------|--------|-------------|
| Range | *r* | Distance to observed object |
| Azimuth | *φ* | Horizontal bearing in node's local frame |
| Elevation | *θ* | Vertical bearing in node's local frame |
| Radial velocity | *vᵣ* | Velocity toward/away from node |

### 3.2 Transformation Pipeline

**Step 1: Compute position in node frame**

```
Position in Nᵢ:
p_Nᵢ = r × [cos(φ)cos(θ), sin(φ)cos(θ), sin(θ)]ᵀ
```

**Step 2: Transform to torso frame**

```
Position in T:
p_T = ᵀTNᵢ × p_Nᵢ
```

The transform ᵀTNᵢ is a 3×3 rotation matrix derived from the relative orientation between the node's IMU and the belt IMU.

**Step 3: Transform velocity**

The radial velocity must account for node motion relative to torso:

```
Velocity in T:
v_T = ᵀTNᵢ × [vᵣ, 0, 0]ᵀ + (∂ᵀTNᵢ/∂t) × p_Nᵢ
```

Where `∂ᵀTNᵢ/∂t` is the angular velocity of the node relative to torso (from IMU gyroscope fusion).

**Step 4: Position translation correction (optional)**

If high precision required, add translation from node position to torso origin:

```
p_T_corrected = p_T + t_Nᵢ_to_T
```

Where `t_Nᵢ_to_T` is the geometric offset of node *i* from the belt node (estimated during calibration).

For most sensing modalities, this translation is negligible (centimeters) compared to observation range (meters), but becomes relevant for:
- Close-range obstacle detection (anklet ToF at 0.3 m)
- Dense SLAM mapping
- High-precision trajectory prediction

---

## 4. The Node-to-Torso Transform (ᵀTNᵢ)

### 4.1 IMU-Based Estimation

Each node has an IMU that continuously estimates its orientation. The node-to-torso transform is computed from the relative orientation:

```
ᵀTNᵢ = (ⁱq_Belt)⁻¹ × ⁱq_Node
```

Where `ⁱq_Node` is the quaternion from node *i*'s IMU, and `ⁱq_Belt` is the quaternion from the belt IMU.

### 4.2 IMU Fusion Algorithm

Each IMU runs a local sensor fusion algorithm:

| Algorithm | Complexity | Drift Characteristics | Use Case |
|-----------|------------|----------------------|----------|
| Madgwick | Low | Moderate drift | Standard nodes |
| Mahony | Low | Similar to Madgwick | Resource-constrained nodes |
| EKF (Extended Kalman Filter) | Medium | Lower drift | Belt node, 360° pendant |
| UKF (Unscented Kalman Filter) | High | Best drift | Research, high-precision |

**Implementation in OMNI-SENSE:**

```rust
// Each node runs this in firmware
pub struct NodeIMUFusion {
    madgwick: MadgwickFilter,
    gyro_bias: Vector3<f32>,
    accel_bias: Vector3<f32>,
    mag_available: bool,
}

impl NodeIMUFusion {
    pub fn update(&mut self, gyro: Vector3<f32>, accel: Vector3<f32>, mag: Option<Vector3<f32>>, dt: f32) -> Quaternion<f32> {
        // Remove biases
        let gyro_unbiased = gyro - self.gyro_bias;
        let accel_unbiased = accel - self.accel_bias;
        
        // Run fusion
        self.madgwick.update(gyro_unbiased, accel_unbiased, mag, dt);
        
        self.madgwick.orientation()
    }
}
```

### 4.3 Multi-IMU Constraint Fusion

The body is a kinematic chain. Adjacent segments have known constraints:

**Anatomical Constraints:**

| Segment Pair | Joint | DOF | Constraint |
|--------------|-------|-----|------------|
| Torso ↔ Head | Neck | 3 rotation | Max ~90° rotation in any axis |
| Torso ↔ Upper Arm | Shoulder | 3 rotation | Ball-and-socket, wide range |
| Upper Arm ↔ Lower Arm | Elbow | 1 rotation | Hinge joint, ~150° range |
| Torso ↔ Thigh | Hip | 3 rotation | Ball-and-socket |
| Thigh ↔ Lower Leg | Knee | 1 rotation | Hinge joint, ~140° range |
| Lower Leg ↔ Foot | Ankle | 2 rotation | ~45° in each axis |

**Constraint application in fusion:**

```rust
// Apply kinematic constraints between adjacent IMUs
pub fn apply_kinematic_constraints(
    torso_orientation: Quaternion<f32>,
    head_orientation: Quaternion<f32>,
    constraint: NeckConstraint
) -> Quaternion<f32> {
    // Compute relative orientation
    let relative = torso_orientation.inverse() * head_orientation;
    
    // Check if within anatomical limits
    let (roll, pitch, yaw) = relative.to_euler();
    
    // Clamp to anatomical range
    let clamped = Quaternion::from_euler(
        roll.clamp(-constraint.max_roll, constraint.max_roll),
        pitch.clamp(-constraint.max_pitch, constraint.max_pitch),
        yaw.clamp(-constraint.max_yaw, constraint.max_yaw)
    );
    
    // Reconstruct head orientation
    torso_orientation * clamped
}
```

---

## 5. The Torso-to-World Transform (ᵂTT)

### 5.1 Why This Transform Matters

The torso-to-world transform determines how the body frame is oriented in the world. Two use cases:

**World-frame sensing (navigation, mapping):**
- Need to know absolute orientation
- Magnetometer provides heading reference
- GPS or visual landmarks provide position

**Body-frame sensing (threat awareness, alerts):**
- Only need relative orientation
- Heading drift is acceptable
- Absolute position not required

### 5.2 Estimation Methods

| Method | Accuracy | Drift | Requirements |
|--------|----------|-------|--------------|
| IMU-only | Poor heading | High | None |
| IMU + Magnetometer | Moderate | Low | No magnetic disturbance |
| IMU + GPS heading | Good | Very low | Outdoor, GPS signal |
| IMU + Visual SLAM | Excellent | Very low | Camera, compute |
| IMU + UWB ranging | Good | Low | UWB anchors or known positions |

### 5.3 Default Approach: Translation-Only Stabilization

For threat awareness (the primary SENTINEL-WEAR use case), the system uses **translation-only stabilization**:

```
Stabilized Torso Frame (S):
├── Origin: Tracks wearer's translation in world
└── Orientation: FIXED (does not rotate with torso)
```

**Effect:** When the wearer turns, the world frame does NOT rotate. An object that was "to the right" remains "to the right" even after the wearer turns 180°.

**Implementation:**

```rust
pub struct TranslationOnlyStabilizer {
    world_position: Vector3<f32>,      // Tracks wearer's position in world
    initial_heading: f32,               // Heading at initialization
}

impl TranslationOnlyStabilizer {
    pub fn stabilize(&self, torso_detection: Detection) -> Detection {
        // Position: translate to world coordinates
        let world_position = self.world_position + torso_detection.position;
        
        // Orientation: use initial heading (don't rotate)
        let stabilized_bearing = torso_detection.bearing + self.initial_heading;
        
        Detection {
            position: world_position,
            bearing: stabilized_bearing,
            velocity: torso_detection.velocity,  // Velocity is already in torso frame
            ..torso_detection
        }
    }
}
```

### 5.4 Stabilization Mode Selection

| Mode | Use Case | Latency | Heading Drift Impact |
|------|----------|---------|---------------------|
| **Translation-only (default)** | Threat awareness, alerts | Lowest | Irrelevant |
| **Full world-frame** | Navigation, mapping | Higher | Critical |
| **No stabilization** | Diagnostics, calibration | Lowest | N/A |

**Configuration:**

```toml
[body_frame.stabilization]
mode = "translation_only"    # "translation_only" | "world_frame" | "none"
heading_source = "initial"   # "initial" | "magnetometer" | "gps" | "visual_slam"
```

---

## 6. Drift Correction and Time Synchronization

### 6.1 Clock Drift Between Nodes

Each node has its own crystal oscillator, causing clock drift. For coherent fusion, all nodes must share a common time reference.

**Problem:** A detection at time *t* from node A may be compared to a detection at time *t + Δ* from node B due to clock offset.

**Solution: BAN time synchronization**

The belt node acts as the **time master**. All other nodes synchronize to the belt's clock.

### 6.2 BLE-Based Time Synchronization

```
Time Sync Exchange (every connection event):

Belt → Node:     t1 (belt transmit timestamp)
Node receives:   t2 (node local receive time)
Node → Belt:     t2, t3 (node local time at response transmit)
Belt receives:   t4 (belt local receive timestamp)

Clock offset estimation:
offset = ((t2 - t1) + (t3 - t4)) / 2

Round-trip time (for filtering outliers):
rtt = (t4 - t1) - (t3 - t2)
```

**Implementation:**

```rust
pub struct ClockOffsetEstimator {
    offset: f32,
    drift_rate: f32,
    last_sync_ms: u64,
}

impl ClockOffsetEstimator {
    pub fn update(&mut self, t1: u64, t2: u64, t3: u64, t4: u64) {
        let rtt = (t4 - t1) as f32 - (t3 - t2) as f32;
        
        // Filter out outliers (high RTT indicates interference)
        if rtt > self.max_rtt_ms {
            return;  // Skip this sync
        }
        
        // Estimate offset
        let new_offset = ((t2 - t1) as f32 - (t4 - t3) as f32) / 2.0;
        
        // Track drift rate
        let dt = (t4 - self.last_sync_ms) as f32 / 1000.0;
        self.drift_rate = (new_offset - self.offset) / dt;
        
        self.offset = new_offset;
        self.last_sync_ms = t4;
    }
    
    pub fn synchronize(&self, node_timestamp: u64) -> u64 {
        // Convert node local time to belt time
        let belt_time = node_timestamp as f32 + self.offset;
        belt_time as u64
    }
}
```

### 6.3 UWB-Based Time Synchronization

For higher precision (sub-nanosecond), UWB provides superior time sync:

| Synchronization Method | Precision | Power Cost | Nodes Required |
|------------------------|-----------|------------|----------------|
| BLE connection events | ~1-10 ms | Low | All nodes (BLE mandatory) |
| BLE with offset tracking | ~0.1-1 ms | Low | All nodes |
| UWB ranging | ~0.1-1 µs | Moderate | Nodes with UWB |
| UWB precise ranging | < 1 ns | Higher | Nodes with UWB |

**When UWB time sync is needed:**
- Extreme velocity detection (microsecond timing)
- Dense SLAM (coherent multi-camera stitching)
- High-precision gait analysis (anklet-to-anklet timing)

**Configuration:**

```toml
[time_synchronization]
primary_source = "ble"               # "ble" | "uwb"
sync_interval_ms = 100               # Sync every 100 ms
max_clock_drift_ppm = 50             # Parts per million
uwb_precision_mode = false           # Enable for extreme velocity
```

---

## 7. Head-Torso Separation (Eyewear IMU)

### 7.1 The Problem

The eyewear node sits on the wearer's head. The head can rotate independently of the torso:
- Looking left/right while walking forward
- Looking up/down while torso faces forward
- Head tilt for any reason

**If not handled correctly:** Detections from the eyewear node are attributed to the direction the head is facing, not the direction the torso is facing. This causes directional alert errors.

### 7.2 Head Frame Definition

```
Head Frame (H) — Eyewear IMU Reference
├── Origin: Eyewear node position (between eyes)
├── +X axis: Wearer's right
├── +Y axis: Where the wearer is LOOKING (gaze direction)
└── +Z axis: Up (opposite gravity, through top of head)
```

### 7.3 Head-Torso Transform (ᵀTH)

The transform from head frame to torso frame is estimated from the relative orientation of the eyewear IMU and belt IMU:

```
ᵀTH = (ᵗq_Belt)⁻¹ × ʰq_Eyewear
```

### 7.4 Transforming Eyewear Detections

```rust
pub fn transform_eyewear_detection(
    head_detection: Detection,
    head_to_torso: Rotation3<f32>,
    head_to_world: Rotation3<f32>,
    stabilization_mode: StabilizationMode
) -> Detection {
    match stabilization_mode {
        // Detection direction is where head was looking, not torso direction
        StabilizationMode::TranslationOnly => {
            // Transform from head frame to torso frame
            let torso_position = head_to_torso * head_detection.position;
            let torso_velocity = head_to_torso * head_detection.velocity;
            
            Detection {
                position: torso_position,
                velocity: torso_velocity,
                bearing: compute_bearing(torso_position),
                ..head_detection
            }
        }
        
        // For world-frame, use head-to-world transform
        StabilizationMode::WorldFrame => {
            let world_position = head_to_world * head_detection.position;
            Detection {
                position: world_position,
                ..head_detection
            }
        }
    }
}
```

### 7.5 Practical Effect

**Scenario:** Wearer is walking forward (torso facing north) but has turned head to look west.

| Detection From | Naive Bearing (head) | Corrected Bearing (torso) |
|----------------|---------------------|--------------------------|
| Eyewear forward camera | West (where head is looking) | **North** (corrected for head turn) |
| Pendant | North | North |
| Bracelets | North | North |

**Correct behavior:** An alert triggered by eyewear detection fires on the bracelet nearest to the actual threat direction (north), not where the wearer was looking (west).

---

## 8. 360° Curved Pendant Contribution

### 8.1 Continuous Omnidirectional Sensing

The 360° curved pendant provides continuous coverage from the chest position:
- 8 cameras at 45° intervals
- mmWave radar arrays at multiple angular positions
- Distributed microphone array

**This is geometrically superior to single-direction sensing:**
- No blind spots in lateral directions
- Seamless tracking through angular transitions
- Detection of objects behind the wearer

### 8.2 Pendant Frame Definition

```
Pendant Frame (P) — 360° Pendant Reference
├── Origin: Pendant center
├── +X axis: Wearer's right
├── +Y axis: Wearer's forward
└── +Z axis: Up
```

The pendant frame is nearly aligned with the torso frame (both at waist/chest level). The transform ᵀTP is small and primarily represents:
- Pendant rotation on the chain (small ±10° swing)
- Pendant sway during walking (pendulum motion)

### 8.3 Pendant Sway Compensation

During walking, the pendant swings like a pendulum. This creates motion blur in cameras and apparent motion in radar.

**Compensation approach:**

```rust
pub struct PendantSwayCompensator {
    pendulum_frequency_hz: f32,      // Natural frequency of pendant swing
    max_sway_angle_deg: f32,         // Maximum expected sway angle
    sway_state: PendulumState,
}

impl PendantSwayCompensator {
    pub fn compensate(&mut self, pendant_imu: &IMUReading) -> Rotation3<f32> {
        // Estimate sway angle from IMU pendulum dynamics
        // Pendulum equation: θ'' = -(g/L) * sin(θ)
        // Simplified: use IMU accelerometer to estimate angle
        
        let gravity_vector = pendant_imu.accel.normalize();
        let sway_angle = gravity_vector.z.acos();  // Angle from vertical
        
        // Compute rotation to "undo" the sway
        let rotation_axis = Vector3::z_axis().cross(&gravity_vector);
        let compensation = Rotation3::from_axis_angle(&rotation_axis, -sway_angle);
        
        compensation
    }
}
```

### 8.4 360° Coverage in Body Frame

The 360° pendant's observations are already in a near-torso-aligned frame. The key transformation is:

```
For each camera k in 8-camera array:
├── Camera k's angular position: α_k = k × 45° (0° = forward)
├── Detection in camera frame: (r, φ, θ)_k
├── Transform to pendant frame:
│   p_P = [r·cos(α_k + φ)·cos(θ), r·sin(α_k + φ)·cos(θ), r·sin(θ)]
├── Apply sway compensation
├── Result: detection in stabilized pendant frame ≈ torso frame
```

---

## 9. Node-Specific Drift Profiles

### 9.1 Why Drift Profiles Differ

Each node type has different motion characteristics affecting IMU drift:

| Node | Motion Characteristics | Drift Drivers |
|------|----------------------|---------------|
| Belt | Relatively stable, rotates with torso | Long-term bias drift |
| Pendant | Sway, rotation | Pendulum dynamics, swing acceleration |
| Bracelet | High angular variability, arm swing | Frequent large accelerations, gesture motion |
| Anklet | Cyclic motion, heel strike impacts | High-G impacts, gait cycle |
| Eyewear | Head rotation independent of torso | Neck movement, gaze shifts |

### 9.2 Drift Profile Parameters

```rust
pub struct DriftProfile {
    pub bias_instability_deg_per_hour: f32,
    pub angle_random_walk_deg_per_sqrt_hour: f32,
    pub max_angular_rate_deg_per_sec: f32,
    pub impact_threshold_g: f32,
    pub recovery_time_after_impact_ms: u32,
}

pub const NODE_DRIFT_PROFILES: [DriftProfile; 5] = [
    // Belt
    DriftProfile {
        bias_instability_deg_per_hour: 3.0,
        angle_random_walk_deg_per_sqrt_hour: 0.1,
        max_angular_rate_deg_per_sec: 180.0,
        impact_threshold_g: 2.0,
        recovery_time_after_impact_ms: 100,
    },
    // Pendant
    DriftProfile {
        bias_instability_deg_per_hour: 5.0,
        angle_random_walk_deg_per_sqrt_hour: 0.15,
        max_angular_rate_deg_per_sec: 90.0,
        impact_threshold_g: 1.5,
        recovery_time_after_impact_ms: 50,
    },
    // Bracelet
    DriftProfile {
        bias_instability_deg_per_hour: 10.0,
        angle_random_walk_deg_per_sqrt_hour: 0.3,
        max_angular_rate_deg_per_hour: 540.0,
        impact_threshold_g: 3.0,
        recovery_time_after_impact_ms: 200,
    },
    // Anklet
    DriftProfile {
        bias_instability_deg_per_hour: 15.0,
        angle_random_walk_deg_per_sqrt_hour: 0.5,
        max_angular_rate_deg_per_sec: 360.0,
        impact_threshold_g: 10.0,  // Heel strike
        recovery_time_after_impact_ms: 50,
    },
    // Eyewear
    DriftProfile {
        bias_instability_deg_per_hour: 5.0,
        angle_random_walk_deg_per_sqrt_hour: 0.15,
        max_angular_rate_deg_per_sec: 300.0,
        impact_threshold_g: 2.0,
        recovery_time_after_impact_ms: 100,
    },
];
```

### 9.3 Impact on PentaTrack

PentaTrack's drift profiles for tracked objects are tuned based on which node detected them:

```rust
pub fn select_drift_profile(detecting_node: NodeType) -> PentaTrackDriftProfile {
    match detecting_node {
        NodeType::Belt => PentaTrackDriftProfile {
            position_noise_sigma: 0.3,
            velocity_noise_sigma: 0.1,
            prune_threshold: 0.1,
        },
        NodeType::Bracelet => PentaTrackDriftProfile {
            position_noise_sigma: 0.6,  // Higher due to arm motion
            velocity_noise_sigma: 0.2,
            prune_threshold: 0.15,
        },
        NodeType::Anklet => PentaTrackDriftProfile {
            position_noise_sigma: 0.5,
            velocity_noise_sigma: 0.15,
            prune_threshold: 0.12,
        },
        // ...
    }
}
```

---

## 10. PentaTrack Integration

### 10.1 PentaTrack in the Stabilized Body Frame

PentaTrack maintains predictive center fields for tracked entities. In SENTINEL-WEAR:

**PentaTrack's "world frame" = Stabilized body frame (S)**

This means:
- Prediction centers are computed in body-relative coordinates
- Drift analysis is performed against body-frame motion patterns
- Intercept projections (if used) are in body-relative terms

### 10.2 Track Lifecycle

```
1. Detection arrives from node i
   └── Timestamp: synchronized to belt clock
   └── Position: transformed to torso frame (T)

2. Body-frame fusion
   └── Combine with other nodes' detections
   └── JPDA association
   └── Covariance intersection fusion

3. Stabilization
   └── Apply translation-only stabilization
   └── Result: detection in stabilized frame (S)

4. PentaTrack update
   └── Assign to existing track OR create new track
   └── Update prediction centers
   └── Compute drift analysis
   └── Generate anomaly flags if drift unusual

5. Alert decision
   └── Check track against alert zones
   └── Compute haptic direction
   └── Route alert to appropriate node
```

### 10.3 Track Representation

```rust
pub struct BodyFrameTrack {
    pub id: TrackId,
    pub position: Vector3<f32>,           // In stabilized body frame
    pub velocity: Vector3<f32>,           // In stabilized body frame
    pub prediction_centers: Vec<PredictionCenter>,
    pub classification: TrackClass,
    pub confidence: f32,
    pub drift_metrics: DriftMetrics,
    pub last_update: u64,
    pub detecting_nodes: Vec<NodeId>,     // Which nodes contributed
}
```

### 10.4 Prediction Centers in Body Frame

Prediction centers represent where the tracked entity is predicted to be:

```rust
pub struct PredictionCenter {
    pub position: Vector3<f32>,           // Predicted position
    pub time_offset_ms: f32,              // Time from now
    pub confidence: f32,
    pub based_on_nodes: Vec<NodeId>,
}
```

**For extreme velocity detection:** Prediction centers are computed for very short time horizons (milliseconds) to project the trajectory of fast-moving objects.

---

## 11. Calibration

### 11.1 Calibration Objectives

Calibration establishes:
1. Each node's geometric position on the body
2. Each node's neutral orientation relative to torso
3. IMU biases and scale factors
4. Inter-node timing offsets
5. 360° pendant camera extrinsics (if applicable)

### 11.2 Neutral-Pose Calibration

**Procedure:**

1. Wearer stands in neutral pose:
   - Feet hip-width apart
   - Arms relaxed at sides
   - Head facing forward
   - Torso upright

2. System records:
   - Each node's IMU orientation → designated as neutral
   - Relative orientations → mounted transform estimates

3. Duration: 5-10 seconds of stable pose

**Implementation:**

```rust
pub struct NeutralPoseCalibrator {
    samples: Vec<CalibrationSample>,
    required_samples: usize,
    stability_threshold_deg: f32,
}

impl NeutralPoseCalibrator {
    pub fn collect(&mut self, node_readings: &HashMap<NodeId, IMUReading>) -> CalibrationStatus {
        // Check stability (all nodes should be nearly stationary)
        for (node_id, reading) in node_readings {
            let angular_rate = reading.gyro.norm();
            if angular_rate > self.stability_threshold_deg {
                return CalibrationStatus::Unstable;
            }
        }
        
        // Record sample
        self.samples.push(CalibrationSample {
            timestamp: current_time_ms(),
            orientations: node_readings.iter()
                .map(|(id, r)| (*id, r.orientation))
                .collect(),
        });
        
        if self.samples.len() >= self.required_samples {
            CalibrationStatus::Complete(self.compute_transforms())
        } else {
            CalibrationStatus::Collecting(self.samples.len(), self.required_samples)
        }
    }
}
```

### 11.3 Walk-Through Calibration

**Purpose:** Estimate geometric node positions using motion as a signal.

**Procedure:**

1. After neutral-pose calibration, wearer walks 20-30 steps at normal pace
2. Each node detects the wearer's own motion (from IMU)
3. System trilaterates node positions from motion signatures
4. Refines mounting transforms

**Algorithm:**

```
For each node i:
    Track wearer's motion pattern over 20 steps
    Compute position of node on body from:
    - Timing of heel strikes (anklet nodes have strongest signal)
    - Arm swing phase (bracelet nodes)
    - Torso translation (belt and pendant)

Cross-reference all nodes:
    Ankle L and Ankle R positions → leg separation
    Bracelet L and R positions → arm separation
    Belt and Pendant positions → torso center
```

### 11.4 360° Pendant Camera Calibration

The 360° pendant requires additional calibration:

**Intrinsic calibration:** Lens distortion parameters for each camera

**Extrinsic calibration:** Relative position and orientation of each camera in the pendant arc

**Stitching calibration:** Overlap regions and seam placement

**Procedure:**

1. Print ArUco marker board
2. Hold board at multiple distances and angles
3. Capture frames from all 8 cameras simultaneously
4. Compute:
   - Intrinsic parameters per camera
   - Extrinsic parameters relative to pendant center
   - Stitching transform

**Configuration output:**

```json
{
  "pendant_360_calibration": {
    "cameras": [
      {
        "id": 0,
        "intrinsics": {"fx": 350.2, "fy": 350.1, "cx": 320.0, "cy": 240.0},
        "distortion": [0.02, -0.01, 0.0, 0.0, 0.0],
        "extrinsics": {
          "rotation": [1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0],
          "translation": [0.0, 0.0, 0.0]
        }
      }
      // ... cameras 1-7
    ],
    "stitching": {
      "resolution": [3840, 1920],
      "seam_positions_deg": [22.5, 67.5, 112.5, 157.5, 202.5, 247.5, 292.5, 337.5]
    }
  }
}
```

### 11.5 Calibration Confidence and Recalibration

The system tracks calibration confidence:

| Confidence | Meaning | Action |
|------------|---------|--------|
| > 0.90 | Excellent | No action needed |
| 0.75 - 0.90 | Good | Normal operation |
| 0.50 - 0.75 | Marginal | Suggest recalibration |
| < 0.50 | Poor | Prompt recalibration required |

**Confidence degradation causes:**
- Extended wear without recalibration
- Node repositioning
- Significant temperature change
- Impacts or drops
- Battery replacement

---

## 12. Failure Modes and Degradation

### 12.1 Magnetic Disturbance

**Cause:** Strong magnetic fields (speakers, motors, metal structures)

**Effect:** Magnetometer-based heading drifts

**Impact by stabilization mode:**

| Stabilization Mode | Impact |
|-------------------|--------|
| Translation-only | **None** (heading not used) |
| World-frame | **Severe** (position errors in world coordinates) |

**Mitigation:**
- Downweight magnetometer in disturbed environments
- Rely on IMU-only drift correction for translation-only mode
- Alert user if world-frame mode is affected

### 12.2 Node Loss

**Cause:** Battery depletion, hardware failure, removed from body

**Effect:** Kinematic chain broken, constraints lost

**Degradation:**

| Nodes Lost | Impact | Degraded Capability |
|------------|--------|---------------------|
| One bracelet | Minor | Reduced lateral sensing |
| One anklet | Moderate | Reduced gait accuracy |
| Pendant | Major | Loss of primary sensing |
| Belt | **Critical** | System non-functional |
| Eyewear | Minor | No impact (optional node) |

**Mitigation:**
- Continue operation with remaining nodes
- Report degraded coverage to user
- Adjust fusion weights to favor healthy nodes

### 12.3 Extreme Motion

**Cause:** Vehicle crash, fall, intense exercise

**Effect:** IMU saturation, assumption violation

**Detection:**
- Angular rate exceeds threshold
- Acceleration exceeds threshold
- Divergence between nodes exceeds threshold

**Mitigation:**
- Switch to degraded mode
- Increase detection thresholds
- Reject observations during extreme motion
- Flag affected data in recordings

### 12.4 Calibration Drift

**Cause:** Long wear sessions, temperature change

**Effect:** Node transforms become inaccurate

**Detection:**
- Cross-node observation disagreement
- IMU bias drift exceeding threshold

**Mitigation:**
- Prompt user to recalibrate
- Attempt automatic recalibration during stable periods

---

## 13. Integration with Extreme Velocity Detection

### 13.1 Latency Requirements

| Scenario | Time Budget | Latency Contribution from Body-Frame |
|----------|-------------|--------------------------------------|
| Rifle (850 m/s, 3 m) | 3.5 ms total | < 0.5 ms for transform |
| Handgun (350 m/s, 3 m) | 8.5 ms total | < 0.5 ms for transform |

### 13.2 Fast-Path Transform

For extreme velocity detection, a simplified transform is used:

```rust
// Simplified fast transform for extreme velocity
// Skips full optimization in favor of speed
pub fn fast_transform_detection(
    node_detection: &RadarDetection,
    node_to_torso: &Rotation3<f32>,
    belt_time_offset: f32
) -> BodyFrameDetection {
    // Minimal computation
    let position = node_to_torso * node_detection.position;
    let velocity = node_to_torso * node_detection.velocity;
    
    // No covariance propagation, no filtering
    // Just geometry for immediate trajectory estimation
    
    BodyFrameDetection {
        position,
        velocity,
        bearing: position.x.atan2(position.y),
        range: position.norm(),
        timestamp: node_detection.timestamp + belt_time_offset as u64,
    }
}
```

### 13.3 QoS Integration

Extreme velocity detections use **QoS = Critical**:
- Bypass normal scheduling
- Immediate transmission
- Preempt other traffic

This ensures the body-frame transform result reaches alert routing within milliseconds.

---

## 14. Configuration Summary

```toml
[body_frame]
# Stabilization
stabilization_mode = "translation_only"
heading_source = "initial"

# IMU fusion
imu_fusion_algorithm = "madgwick"
kinematic_constraints_enabled = true

# Calibration
auto_recalibration = true
recalibration_confidence_threshold = 0.7

# Time synchronization
time_sync_source = "ble"
time_sync_interval_ms = 100

[body_frame.drift_profiles]
# Override defaults if needed
belt_bias_instability_deg_per_hour = 3.0
pendant_bias_instability_deg_per_hour = 5.0
bracelet_bias_instability_deg_per_hour = 10.0
anklet_bias_instability_deg_per_hour = 15.0

[body_frame.360_pendant]
sway_compensation_enabled = true
max_sway_angle_deg = 10.0
```

---

## 15. Civilian Transfer

The body-coordinate-fusion architecture transfers directly to:

| Domain | Application | Key Difference |
|--------|-------------|----------------|
| **Sports biomechanics** | Multi-IMU pose estimation for technique analysis | Higher precision requirements, often offline processing |
| **Accessibility wearables** | Directional haptic alerts for visually impaired | Translation-only stabilization is essential |
| **Physical therapy** | Motion pattern characterization | Clinical validation required |
| **Research gait analysis** | Multi-IMU gait reconstruction | Research-grade accuracy, open data |
| **Industrial wearables** | Worker safety awareness | Different threat model, similar fusion |
| **VR/AR** | Body tracking for virtual environments | Lower latency, higher precision |

The fusion architecture is the same; the perception-field consumer changes.

---

## 16. API Reference

### 16.1 Body-Frame Fusion Interface

```rust
pub trait BodyFrameFusion {
    /// Transform a node-local detection to body frame
    fn transform_detection(&self, detection: NodeDetection) -> Result<BodyFrameDetection, FusionError>;
    
    /// Fuse multiple node detections into single track
    fn fuse_detections(&mut self, detections: Vec<NodeDetection>) -> Option<FusedTrack>;
    
    /// Get current torso orientation
    fn torso_orientation(&self) -> Quaternion<f32>;
    
    /// Get node-to-torso transform for a specific node
    fn node_transform(&self, node_id: NodeId) -> Option<Rotation3<f32>>;
    
    /// Get calibration confidence
    fn calibration_confidence(&self) -> f32;
    
    /// Trigger recalibration
    fn recalibrate(&mut self) -> CalibrationStatus;
}
```

### 16.2 Stabilization Interface

```rust
pub trait Stabilizer {
    /// Stabilize a body-frame detection
    fn stabilize(&self, detection: BodyFrameDetection) -> StabilizedDetection;
    
    /// Get current stabilization mode
    fn mode(&self) -> StabilizationMode;
    
    /// Set stabilization mode
    fn set_mode(&mut self, mode: StabilizationMode);
}
```

---

**End of Body-Coordinate Fusion Documentation**
