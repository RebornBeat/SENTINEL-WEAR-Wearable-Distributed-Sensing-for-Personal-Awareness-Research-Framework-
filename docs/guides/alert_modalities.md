# Alert Modalities — SENTINEL-WEAR

**Version:** 0.2 | **Status:** Research Reference Design

---

## 1. The Design Principle: Ambient Awareness Without Intrusion

SENTINEL-WEAR alerts are designed to be ambient — felt rather than heard, providing directional context without interrupting normal activity. The primary modality is haptic (vibration); secondary modalities are audio (optional bone conduction) and visual (companion app).

**Core philosophy:**
- Alerts inform, not alarm
- Direction is intuitive (body-part-based, not compass-based)
- Urgency is encoded in pattern, not volume
- User maintains full control over all alert behavior

**No alert is automatic for anything except the standard alert classes.** Emergency contact notification is always manually triggered.

---

## 2. Alert Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ALERT GENERATION PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  Detection Event (any node)                                                  │
│       │                                                                      │
│       ▼                                                                      │
│  Belt Node Fusion Engine (PentaTrack + Classification)                       │
│       │                                                                      │
│       ├── Alert Classification                                               │
│       │   ├── Class: HumanApproaching, VehicleNear, etc.                    │
│       │   ├── Urgency: Info / Warning / Critical                            │
│       │   └── Direction: relative to body frame                             │
│       │                                                                      │
│       ▼                                                                      │
│  Alert Router (Belt Node)                                                    │
│       │                                                                      │
│       ├── Haptic Route → Target node(s) based on direction                  │
│       │   └── Pattern selected based on class + urgency                     │
│       │                                                                      │
│       ├── Audio Route (if enabled) → Bone conduction earpiece               │
│       │                                                                      │
│       ├── App Route (if connected) → WebSocket to companion app             │
│       │                                                                      │
│       └── Cellular Route (if remote) → Push notification                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Alert Classes and Priority

### 3.1 Alert Class Hierarchy

| Class | Urgency | Auto-Alert | Description |
|-------|---------|------------|-------------|
| **FastObjectDetected** | Critical | ✅ Yes | Extreme velocity projectile detection |
| **FallDetected** | Critical | ✅ Yes | Fall event detected (impact + orientation change) |
| **StumblePrecursor** | Warning | ✅ Yes | Gait anomaly indicating imminent fall |
| **GaitAnomaly** | Warning | ✅ Yes | Irregular gait pattern detected |
| **HumanApproaching Fast** | Warning | ✅ Yes | Human approaching > 2 m/s relative |
| **HumanApproaching Slow** | Info | ✅ Yes | Human approaching < 1 m/s relative |
| **VehicleNear** | Warning | ✅ Yes | Vehicle detected in proximity |
| **PetApproaching** | Info | ⚠️ Configurable | Pet movement detected |
| **UnknownObject** | Info | ✅ Yes | Unidentified movement detected |
| **NodeOffline** | Warning | ✅ Yes | Sensor node lost connection |
| **BatteryLow** | Warning | ✅ Yes | Any node battery below threshold |
| **CalibrationRequired** | Info | ✅ Yes | Body-frame calibration degraded |

### 3.2 QoS Classes for Alert Transmission

Alerts are transmitted over BLE with QoS priority to ensure timely delivery:

| QoS Class | Priority | Latency Target | Alert Classes |
|-----------|----------|----------------|---------------|
| **Critical** | Highest | < 5 ms | FastObjectDetected, FallDetected |
| **High** | Second | < 20 ms | StumblePrecursor, GaitAnomaly |
| **Medium** | Third | < 50 ms | HumanApproaching, VehicleNear |
| **Low** | Fourth | < 500 ms | BatteryLow, CalibrationRequired |

**Critical QoS behavior:**
- Preempts all other BLE traffic
- Transmitted immediately, regardless of scheduled slot
- Guaranteed delivery priority
- Used for extreme velocity detection and fall events

**Configuration:**

```toml
[alerts.qos]
critical_preempt_all = true
critical_max_latency_ms = 5
high_max_latency_ms = 20
medium_max_latency_ms = 50
low_max_latency_ms = 500
```

---

## 4. Haptic Alert System

### 4.1 Directional Encoding

The haptic actuator that fires indicates the direction of the detected object relative to the wearer's body. The wearer does not need to interpret a compass direction — they feel the alert on the relevant body part.

**This is intuitive body-part mapping:**

| Approach Direction | Primary Haptic Node | Secondary Haptic Node | Intuitive Meaning |
|---|---|---|---|
| Front | Pendant or belt | — | "Something in front of you" |
| Front-left | Left bracelet | Pendant | "Something from front-left" |
| Left | Left bracelet | Left anklet | "Something from your left" |
| Rear-left | Left anklet | Left bracelet | "Something from rear-left" |
| Rear | Both anklets simultaneously | — | "Something behind you" |
| Rear-right | Right anklet | Right bracelet | "Something from rear-right" |
| Right | Right bracelet | Right anklet | "Something from your right" |
| Front-right | Right bracelet | Pendant | "Something from front-right" |
| Above | Pendant | Eyewear (if present) | "Something above you" |
| Below (ground) | Both anklets simultaneously | — | "Ground-level obstacle/hazard" |

**How it works:**
- **Primary node** fires at full intensity
- **Secondary node** fires at 50% intensity to indicate intermediate direction
- User naturally turns toward the haptic source
- No cognitive translation needed (no "compass bearings" to decode)

### 4.2 Pattern Encoding

Haptic patterns encode alert class and urgency:

| Pattern | Duration | Meaning | Urgency |
|---|---|---|---|
| Single pulse | 300 ms | HumanApproaching slow (< 1 m/s relative) | Info |
| Double pulse | 100 ms + 200 ms gap + 100 ms | HumanApproaching fast (> 2 m/s relative) | Warning |
| Triple pulse | 100 ms + 100 ms + 100 ms | VehicleNear | Warning |
| Sustained pulse | 500 ms | GaitAnomaly — stumble precursor | Warning |
| Extended buzz | 1000 ms, repeat after 3s | FallDetected | Critical |
| Rapid pulses | 50 ms on/off × 5 | **FastObjectDetected (extreme velocity)** | **Critical** |
| Slow sweep | 100 ms on, 100 ms off × 3 | CalibrationPrompt | Info |
| Heartbeat pattern | 100 ms + 100 ms + 200 ms gap | Low battery warning | Warning |
| Ascending tones | 50 ms × 3, increasing intensity | NodeOffline | Warning |

### 4.3 Intensity Levels

| Level | Intensity | Use Case |
|---|---|---|
| Full (100%) | Maximum motor drive | Primary alert, Critical/Warning classes |
| Half (50%) | Reduced motor drive | Secondary node in directional encoding |
| Quarter (25%) | Minimum perceptible | Ambient awareness, subtle notifications |
| Adaptive | Variable based on context | Adjusts based on activity level |

**Adaptive intensity configuration:**

```toml
[alerts.haptic.intensity]
default = 1.0
quiet_hours_enabled = true
quiet_hours_start = "22:00"
quiet_hours_end = "07:00"
quiet_hours_intensity = 0.5
active_intensity = 1.0
sleep_mode_intensity = 0.25
```

### 4.4 Haptic Hardware

**Actuator type:** LRA (Linear Resonant Actuator)

| Specification | Value |
|---------------|-------|
| Type | LRA (preferred) or ERM |
| Typical resonance | 150-200 Hz |
| Voltage | 3.0-3.3V |
| Driver IC | TI DRV2605L (I2C, built-in waveform library) |
| Response time | < 50 ms to full vibration |
| Power (active) | 50-100 mA during pulse |

**Driver capabilities:**
- Built-in waveform library (over 100 patterns)
- Closed-loop back-EMF feedback
- Automatic resonance tracking
- Real-time amplitude control

---

## 5. Body-Frame Stabilization and Alert Directionality

### 5.1 The Problem

When the wearer turns their body, what happens to detected tracks?

**Without stabilization:**
- A person approaching from behind appears to "move" to the front when the wearer turns around
- Alerts would fire on the wrong body part
- Confusing and non-intuitive

**With stabilization:**
- Tracks maintain their world-relative direction
- Alerts fire on the correct body part regardless of wearer orientation

### 5.2 Stabilization Modes

SENTINEL-WEAR supports three stabilization modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| **Translational (default)** | Body frame co-translates but does not co-rotate with wearer | Intuitive directional alerts |
| **World-fixed** | Body frame tracks world coordinates (compass-based) | Navigation applications |
| **None** | Raw sensor frame | Diagnostics only |

**Translational stabilization (recommended):**

```
A person approaching from behind (bearing 180°)
    ↓
Wearer turns around to face them
    ↓
Track STAYS at bearing 180° (still "behind" in world frame)
    ↓
Alert fires on anklets (rear direction)
    ↓
Wearer turns back around
    ↓
Alert direction unchanged — still "behind them"
```

This means the alert fires on the node nearest to where the approaching person actually is, regardless of which way the wearer is facing.

### 5.3 Head-Torso Separation

When eyewear is present, head orientation is tracked separately from torso:

```
Torso facing North
Head turned 45° to the right
    ↓
Object detected by eyewear at 0° (forward relative to head)
    ↓
Body-frame transform: 0° + 45° = 45° relative to torso
    ↓
Alert fires on right bracelet (right-front quadrant)
```

This ensures alerts remain correctly attributed regardless of head position.

### 5.4 Configuration

```toml
[alerts.directionality]
stabilization_mode = "translational"  # "translational" | "world_fixed" | "none"
head_torso_separation = true          # Enable when eyewear present
prediction_ahead_seconds = 0.5        # Predict where object will be, not where it was
```

---

## 6. Extreme Velocity Detection Alerts

### 6.1 The FastObjectDetected Alert Class

This is the highest-priority alert in the system, triggered by extreme velocity detection (projectiles, debris, fragments).

**Detection chain:**
```
Doppler radar detects velocity > 50 m/s
    ↓ (within 10-100 µs)
Event camera confirms trajectory
    ↓ (within < 1 ms)
MCU classifies as FastObjectDetected
    ↓ (within 100-500 µs)
Alert routed with QoS = Critical
    ↓ (transmitted immediately, preempting other traffic)
Belt receives alert
    ↓ (within 1-2 ms BLE transmission)
Haptic fires on pendant
    ↓ (within < 1 ms)
Total latency: < 5 ms (target)
```

### 6.2 Pattern for FastObjectDetected

**Pattern:** Rapid pulses — 50 ms on, 50 ms off, repeated 5 times

**Characteristics:**
- Most urgent pattern in the system
- Distinct from all other patterns
- Fires on pendant (primary sensing node)
- Secondary: eyewear (if present, forward detection context)

**Why pendant:**
- Highest information value node
- Center of body frame
- User's attention naturally drawn to torso

### 6.3 Context Information

The FastObjectDetected alert includes trajectory context when available:

```json
{
  "class": "FastObjectDetected",
  "urgency": "Critical",
  "direction": {
    "bearing_deg": 45,
    "elevation_deg": 0
  },
  "velocity_ms": 280,
  "trajectory_prediction": {
    "intersect_body": true,
    "time_to_impact_ms": 4.2
  },
  "confidence": 0.87,
  "node_source": "pendant",
  "timestamp_ns": 1710509422004
}
```

### 6.4 User Response Time

| Projectile Type | Engagement Distance | Time to Impact | Alert Latency (target) | Response Time Available |
|-----------------|---------------------|----------------|------------------------|-------------------------|
| Rifle (850 m/s) | 3 m | 3.5 ms | < 5 ms | 0-2 ms (very limited) |
| Rifle (850 m/s) | 10 m | 11.8 ms | < 5 ms | ~7 ms |
| Handgun (350 m/s) | 3 m | 8.5 ms | < 5 ms | ~3-4 ms |
| Handgun (350 m/s) | 10 m | 28.6 ms | < 5 ms | ~24 ms |

**Key insight:** At very close range (3 m), response time is extremely limited. The alert provides awareness but may not enable avoidance. At longer ranges, the wearer has more time to react.

### 6.5 Configuration

```toml
[alerts.extreme_velocity]
enabled = true
min_velocity_ms = 50                 # Ignore objects below this velocity
max_detection_distance_m = 10        # Only alert for nearby threats
haptic_pattern = "rapid_pulse_5"
haptic_node = "pendant"
secondary_node = "eyewear"
audio_alert = false                  # Optional audio tone
app_notification = true
cellular_notification = false        # Manual only for remote
include_trajectory = true
prediction_ahead = true
```

---

## 7. Fall Detection Alerts

### 7.1 Detection Criteria

Fall detection uses multiple sensor inputs:

| Sensor | Signal | Criteria |
|--------|--------|----------|
| Anklet IMU | Impact acceleration | > 3 g peak (configurable) |
| Anklet IMU | Orientation change | > 60° from vertical within 500 ms |
| Pendant IMU | Orientation change | Torso tilt > 45° |
| Bracelet IMU | Impact deceleration | Secondary confirmation |
| Radar | Vertical velocity | Downward motion > 1 m/s followed by stop |

**Multi-sensor fusion reduces false positives:**
- Single sensor: ~15% false positive rate
- Two sensors: ~5% false positive rate
- Three+ sensors: < 2% false positive rate

### 7.2 FallDetected Alert Pattern

**Pattern:** Extended buzz — 1000 ms continuous vibration

**Behavior:**
- Fires immediately on fall detection
- Repeats every 3 seconds until acknowledged
- Intensity: Full (100%)
- Nodes: All nodes fire simultaneously (full-body awareness)

**Acknowledgment:**
- Wearer presses any button OR
- Companion app "I'm OK" button OR
- Voice command (if microphone active)
- After 60 seconds without acknowledgment: escalation option (configurable)

### 7.3 Fall Detection Does NOT Auto-Call Emergency Contact

**This is by design:**
- False positives could cause unnecessary emergency calls
- Wearer may be conscious but slow to respond
- Emergency services dispatch for false alarms wastes resources
- Legal implications of automated emergency calls vary by jurisdiction

**User responsibility:** Users who need automatic emergency response should use a dedicated medical alert device in addition to SENTINEL-WEAR.

**Configuration:**

```toml
[alerts.fall_detection]
enabled = true
impact_threshold_g = 3.0
orientation_change_deg = 60
min_sensors_required = 2
haptic_pattern = "extended_buzz_1000ms"
haptic_nodes = "all"
repeat_interval_ms = 3000
acknowledgment_timeout_s = 60
escalation_action = "none"          # "none" | "app_notification" | "emergency_contact"
```

---

## 8. Gait Anomaly and Stumble Precursor Alerts

### 8.1 GaitAnomaly Alert

**Triggered when:**
- Step irregularity exceeds threshold
- Stride asymmetry detected
- Unusual foot placement pattern

**Pattern:** Sustained pulse — 500 ms

**Purpose:** Awareness alert — "your walking pattern is unusual"

### 8.2 StumblePrecursor Alert

**Triggered when:**
- Gait phase timing abnormality
- Pre-stumble signature detected from IMU analysis
- Balance instability detected

**Pattern:** Sustained pulse — 500 ms (same as GaitAnomaly, but with higher priority)

**Purpose:** Warning alert — "you may be about to stumble, slow down"

### 8.3 Predictive Capability

The system can warn BEFORE a fall occurs:

```
Normal gait
    ↓
Irregular step detected
    ↓
GaitAnomaly alert fires
    ↓
Balance instability increasing
    ↓
StumblePrecursor alert fires
    ↓
Potential fall trajectory detected
    ↓
FallDetected alert fires (if fall occurs)
```

**Early warning value:** The GaitAnomaly and StumblePrecursor alerts give the wearer time to stop, grab a support, or sit down.

### 8.4 Gait Alert Configuration

```toml
[alerts.gait]
anomaly_enabled = true
stumble_precursor_enabled = true
step_irregularity_threshold = 0.3   # 0-1 scale
asymmetry_threshold = 0.2           # Left/right difference
haptic_pattern = "sustained_500ms"
haptic_nodes = ["anklet_left", "anklet_right"]
intensity = 0.75                    # Noticeable but not alarming
```

---

## 9. Human and Vehicle Presence Alerts

### 9.1 HumanApproaching Alerts

**Two variants:**

| Variant | Criteria | Pattern | Urgency |
|---------|----------|---------|---------|
| Slow | Relative velocity < 1 m/s | Single pulse (300 ms) | Info |
| Fast | Relative velocity > 2 m/s | Double pulse (100+200+100 ms) | Warning |

**Directional firing:** Based on approach bearing relative to body frame.

**Intensity scaling:**
- Closer distance = higher intensity
- Faster velocity = shorter pulses (urgency)
- User can adjust overall sensitivity

### 9.2 VehicleNear Alert

**Pattern:** Triple pulse (100+100+100 ms)

**Characteristics:**
- Distinct from human patterns
- Indicates larger, potentially more dangerous object
- Directional firing based on vehicle bearing

### 9.3 Configuration

```toml
[alerts.presence]
human_slow_enabled = true
human_fast_enabled = true
vehicle_enabled = true
human_slow_velocity_threshold_ms = 1.0
human_fast_velocity_threshold_ms = 2.0
min_distance_m = 0.5               # Ignore if too close (already detected)
max_distance_m = 10.0              # Only alert for nearby presence
intensity_distance_scaling = true   # Closer = stronger
```

---

## 10. Bone-Conduction Audio Alerts (Optional)

### 10.1 Audio Modality

When a bone-conduction earpiece is paired, audio alerts provide supplementary awareness:

| Alert Class | Audio Cue | Tone Character |
|-------------|-----------|----------------|
| HumanApproaching | Bearing-encoded frequency | Higher pitch = more to the right |
| VehicleNear | Distinct vehicle-class tone | Low rumble pattern |
| FallDetected | Persistent tone until acknowledged | Urgent, attention-grabbing |
| FastObjectDetected | Rapid staccato burst | Highest urgency |
| BatteryLow | Battery warning pattern | Ascending tones |
| CalibrationPrompt | Soft chime | Non-urgent notification |

### 10.2 Bearing-Encoded Frequency

For HumanApproaching alerts, the audio tone encodes direction:

```
Bearing: -90° (far left) → 200 Hz
Bearing: 0° (center) → 400 Hz
Bearing: +90° (far right) → 600 Hz

User hears higher pitch = object more to the right
User hears lower pitch = object more to the left
```

### 10.3 Configuration

```toml
[alerts.audio]
enabled = false                     # Opt-in feature
device_type = "bone_conduction"      # "bone_conduction" | "earpiece" | "speaker"
volume = 0.7                         # 0.0 - 1.0
bearing_encoding = true
quiet_hours_enabled = true
quiet_hours_start = "22:00"
quiet_hours_end = "07:00"
```

---

## 11. Companion App Notifications

### 11.1 Live Alert Stream

When the companion app is connected, all alerts are displayed in real-time:

**Alert display includes:**
- Alert class and urgency
- Direction (bearing and elevation)
- Distance estimate
- Confidence
- Timestamp
- Source node

**Visual encoding:**
- Color: Urgency (red=Critical, orange=Warning, blue=Info)
- Icon: Class (human figure, vehicle, hazard symbol)
- Position on radar display: Direction and distance
- Size: Proximity

### 11.2 Historical Alert Log

All alerts are logged and accessible in the companion app:

```
Alert History
├── Today
│   ├── 14:23:05 - FastObjectDetected (Critical) - Front-right
│   ├── 13:45:22 - HumanApproaching Fast (Warning) - Rear
│   ├── 12:01:33 - GaitAnomaly (Warning) - N/A
│   └── 08:15:44 - HumanApproaching Slow (Info) - Left
├── Yesterday
│   └── ...
└── ...
```

### 11.3 Remote Notifications (Cellular)

When the wearer is away from home WiFi and cellular is configured:

| Alert Class | Cellular Push Notification |
|-------------|---------------------------|
| FastObjectDetected | ✅ Yes (if configured) |
| FallDetected | ✅ Yes (if configured) |
| StumblePrecursor | ⚠️ Configurable |
| GaitAnomaly | ❌ No |
| HumanApproaching | ❌ No |
| VehicleNear | ❌ No |
| BatteryLow | ❌ No |

**Configuration:**

```toml
[alerts.remote]
cellular_enabled = true
cellular_classes = ["FastObjectDetected", "FallDetected"]
cellular_min_interval_s = 60        # Don't spam with rapid notifications
include_location = false            # Privacy consideration
include_direction = true
include_velocity = true
```

---

## 12. Emergency Contact Feature

### 12.1 Manual Trigger Only

The emergency contact feature sends a location notification to a pre-configured contact. It is **always manually triggered**:

**Trigger methods:**
1. **Companion app:** Emergency button (long-press, 3 seconds)
2. **Voice command:** "Send emergency alert" (if microphone active)
3. **CLI:** `curl -X POST http://belt-node.local/api/alerts/emergency-contact`
4. **Physical button:** Dedicated emergency button on belt node (if configured)

### 12.2 Emergency Contact Message

When triggered, the emergency contact receives:

```json
{
  "type": "EmergencyAlert",
  "wearer_id": "user_configured_label",
  "timestamp_utc": "2024-03-15T14:30:22.004Z",
  "location": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "accuracy_m": 10
  },
  "battery_percent": 78,
  "alert_history_last_5m": [...],
  "message": "User triggered emergency alert"
}
```

### 12.3 Why Fall Detection Doesn't Auto-Trigger Emergency Contact

**Design rationale:**
1. **False positives:** Even 2% false positive rate would cause unnecessary emergency calls
2. **Conscious wearer:** Many falls don't incapacitate the wearer
3. **Privacy:** Some users don't want location shared automatically
4. **Legal:** Emergency services dispatch rules vary by jurisdiction
5. **Responsibility:** User maintains control over escalation

**User responsibility:** Users who need automatic emergency response for falls should use a dedicated medical alert device (Life Alert, Apple Watch fall detection, etc.) in addition to SENTINEL-WEAR.

### 12.4 Configuration

```toml
[alerts.emergency_contact]
enabled = true
manual_trigger_only = true          # Always true
contact_method = "sms"              # "sms" | "email" | "api_endpoint"
contact_address = "+15551234567"
confirm_before_send = true          # Require confirmation in app
include_location = true
include_recent_alerts = true
include_battery_status = true
```

---

## 13. Alert Routing and Latency

### 13.1 Latency Targets by Alert Class

| Alert Class | QoS Priority | BLE Latency Target | Total Latency Target |
|-------------|--------------|-------------------|----------------------|
| FastObjectDetected | Critical | < 5 ms | < 10 ms |
| FallDetected | Critical | < 5 ms | < 10 ms |
| StumblePrecursor | High | < 20 ms | < 30 ms |
| GaitAnomaly | High | < 20 ms | < 30 ms |
| HumanApproaching Fast | Medium | < 50 ms | < 70 ms |
| HumanApproaching Slow | Medium | < 50 ms | < 70 ms |
| VehicleNear | Medium | < 50 ms | < 70 ms |
| BatteryLow | Low | < 500 ms | < 1 second |
| NodeOffline | Low | < 500 ms | < 1 second |

### 13.2 BLE Slot Reservation for Critical Alerts

To guarantee latency for critical alerts, a dedicated BLE slot is reserved:

```
BLE Connection Event (30 ms interval)
├── Slot 0-2 ms:   Belt → Pendant (sync, commands)
├── Slot 2-5 ms:   Pendant → Belt (sensor data, detections)
├── Slot 5-7 ms:   Belt → Anklet L
├── Slot 7-10 ms:  Anklet L → Belt (gait event)
├── Slot 10-12 ms: Belt → Anklet R
├── Slot 12-15 ms: Anklet R → Belt (gait event)
├── Slot 15-17 ms: Belt → Bracelet L
├── Slot 17-20 ms: Bracelet L → Belt
├── Slot 20-22 ms: Belt → Bracelet R
├── Slot 22-25 ms: Bracelet R → Belt
├── Slot 25-28 ms: RESERVED (Critical alerts — FastObjectDetected, FallDetected)
└── Slot 28-30 ms: Reserved (retransmits)
```

**Critical alerts can also preempt ongoing transmissions** — they don't wait for their slot if the system detects an immediate need.

### 13.3 Antenna Diversity Impact on Alert Reliability

Antenna diversity on the belt node improves alert reliability by:

| Scenario | Single Antenna | Dual Antenna (Diversity) |
|----------|---------------|-------------------------|
| Wearer facing transmitter | Good reliability | Good reliability |
| Wearer facing away from transmitter | Poor reliability (body blocks signal) | Good reliability (other antenna receives) |
| Wearer moving | Variable reliability | Stable reliability |
| Alert during body rotation | May miss slot | More likely to succeed |

**For extreme velocity detection:** Antenna diversity is strongly recommended to ensure the FastObjectDetected alert is transmitted reliably regardless of body position.

### 13.4 Configuration

```toml
[alerts.routing]
ble_qos_enabled = true
critical_slot_reserved = true
critical_slot_start_ms = 25
critical_slot_end_ms = 28
critical_can_preempt = true
antenna_diversity_enabled = true
```

---

## 14. Alert Customization

### 14.1 User-Configurable Settings

All alert behavior is user-configurable:

```toml
[alerts]
# Global settings
enabled = true
quiet_hours_enabled = true
quiet_hours_start = "22:00"
quiet_hours_end = "07:00"

# Modality preferences
haptic_enabled = true
audio_enabled = false
app_notifications = true
cellular_notifications = false

# Per-class configuration
[alerts.classes.FastObjectDetected]
enabled = true
haptic = true
audio = true
app = true
cellular = false
intensity = 1.0

[alerts.classes.FallDetected]
enabled = true
haptic = true
audio = true
app = true
cellular = false
intensity = 1.0

[alerts.classes.HumanApproachingSlow]
enabled = true
haptic = true
audio = false
app = true
cellular = false
intensity = 0.5

[alerts.classes.BatteryLow]
enabled = true
haptic = true
audio = false
app = true
cellular = false
intensity = 0.25
```

### 14.2 Sensitivity Profiles

Pre-configured sensitivity profiles for common use cases:

| Profile | Description | Effect |
|---------|-------------|--------|
| **Maximum Awareness** | All alerts enabled, maximum sensitivity | Every detection triggers alert |
| **Standard** (default) | Balanced sensitivity | Significant events trigger alert |
| **Minimal** | Critical alerts only | Only FallDetected, FastObjectDetected |
| **Sleep** | Quiet hours extended | Only Critical alerts, reduced haptic |
| **Accessibility** | Enhanced for visually impaired | Audio enabled, higher intensity |

### 14.3 Context-Aware Alert Suppression

Alerts can be suppressed based on context:

```toml
[alerts.suppression]
suppress_when_still = false         # Don't alert if wearer is stationary
suppress_when_sitting = false        # Don't alert if posture = sitting
suppress_known_resident = false      # Don't alert for recognized individuals
suppress_same_direction_count = 3    # Don't alert after 3 consecutive same-direction alerts
suppress_interval_s = 10             # Minimum seconds between similar alerts
```

---

## 15. Node-Specific Alert Behavior

### 15.1 Pendant Node

| Alert Class | Haptic | Direction Encoding |
|-------------|--------|-------------------|
| FastObjectDetected | ✅ Primary | Front-centered |
| FallDetected | ✅ All nodes | — |
| HumanApproaching (front) | ✅ Primary | Front |
| HumanApproaching (oblique front) | ✅ Secondary | Front-left/right |

### 15.2 Bracelet Nodes

| Alert Class | Haptic | Direction Encoding |
|-------------|--------|-------------------|
| HumanApproaching (lateral) | ✅ Primary | Left/right |
| HumanApproaching (rear-lateral) | ✅ Secondary | Rear-left/right |
| VehicleNear (lateral) | ✅ Primary | Left/right |

### 15.3 Anklet Nodes

| Alert Class | Haptic | Direction Encoding |
|-------------|--------|-------------------|
| HumanApproaching (rear) | ✅ Primary | Rear |
| HumanApproaching (rear-lateral) | ✅ Primary | Rear-left/right |
| GaitAnomaly | ✅ Primary | — |
| StumblePrecursor | ✅ Primary | — |
| FallDetected | ✅ All nodes | — |

### 15.4 Belt Node

| Alert Class | Haptic | Direction Encoding |
|-------------|--------|-------------------|
| HumanApproaching (front) | ✅ Primary (if no pendant) | Front |
| VehicleNear (front) | ✅ Primary | Front |
| NodeOffline | ✅ Pulse | — |
| BatteryLow | ✅ Pulse | — |

### 15.5 Eyewear Node

| Alert Class | Haptic | Direction Encoding |
|-------------|--------|-------------------|
| FastObjectDetected | ✅ Secondary | Forward context |
| HumanApproaching (front) | ✅ Secondary | Front |

---

## 16. Testing and Calibration

### 16.1 Haptic Test Mode

Test all haptic patterns and nodes:

```bash
# Test all nodes with calibration pattern
curl -X POST http://belt-node.local/api/alerts/test-haptic

# Test specific node
curl -X POST http://belt-node.local/api/alerts/test-haptic \
  -H "Content-Type: application/json" \
  -d '{"node_id": "bracelet_left", "pattern": "double_pulse", "intensity": 1.0}'
```

### 16.2 Alert Simulation

Simulate alert classes for testing:

```bash
curl -X POST http://belt-node.local/api/alerts/simulate \
  -H "Content-Type: application/json" \
  -d '{"class": "FastObjectDetected", "direction": {"bearing_deg": 45, "elevation_deg": 0}}'
```

### 16.3 Latency Measurement

Measure actual alert latency:

```bash
curl -X POST http://belt-node.local/api/alerts/measure-latency \
  -H "Content-Type: application/json" \
  -d '{"class": "FastObjectDetected"}'

# Response:
{
  "detection_timestamp_ns": 1710509422004000,
  "haptic_fire_timestamp_ns": 1710509422008123,
  "latency_us": 4123,
  "latency_target_us": 5000,
  "result": "PASS"
}
```

---

## 17. Summary Configuration

Complete alert configuration example:

```toml
[alerts]
enabled = true
quiet_hours_enabled = true
quiet_hours_start = "22:00"
quiet_hours_end = "07:00"

[alerts.modalities]
haptic = true
audio = false
app_notifications = true
cellular_notifications = false

[alerts.qos]
critical_preempt_all = true
critical_max_latency_ms = 5
high_max_latency_ms = 20
medium_max_latency_ms = 50
low_max_latency_ms = 500
critical_slot_reserved = true
critical_slot_start_ms = 25
critical_slot_end_ms = 28

[alerts.directionality]
stabilization_mode = "translational"
head_torso_separation = true
prediction_ahead_seconds = 0.5

[alerts.haptic]
default_intensity = 1.0
quiet_hours_intensity = 0.5
adaptive_intensity = true

[alerts.audio]
enabled = false
device_type = "bone_conduction"
volume = 0.7
bearing_encoding = true

[alerts.extreme_velocity]
enabled = true
min_velocity_ms = 50
max_detection_distance_m = 10
haptic_pattern = "rapid_pulse_5"
haptic_node = "pendant"
include_trajectory = true

[alerts.fall_detection]
enabled = true
impact_threshold_g = 3.0
min_sensors_required = 2
haptic_pattern = "extended_buzz_1000ms"
haptic_nodes = "all"
repeat_interval_ms = 3000
acknowledgment_timeout_s = 60
escalation_action = "none"

[alerts.gait]
anomaly_enabled = true
stumble_precursor_enabled = true
haptic_pattern = "sustained_500ms"
haptic_nodes = ["anklet_left", "anklet_right"]

[alerts.presence]
human_slow_enabled = true
human_fast_enabled = true
vehicle_enabled = true
human_slow_velocity_threshold_ms = 1.0
human_fast_velocity_threshold_ms = 2.0
intensity_distance_scaling = true

[alerts.remote]
cellular_enabled = true
cellular_classes = ["FastObjectDetected", "FallDetected"]
cellular_min_interval_s = 60

[alerts.emergency_contact]
enabled = true
manual_trigger_only = true
contact_method = "sms"
contact_address = "+15551234567"
include_location = true
```

---

**End of Alert Modalities Guide**
