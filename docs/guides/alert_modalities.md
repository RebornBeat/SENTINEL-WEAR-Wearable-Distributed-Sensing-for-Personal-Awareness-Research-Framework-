# Alert Modalities — SENTINEL-WEAR

---

## The Design Principle: Ambient Awareness Without Intrusion

SENTINEL-WEAR alerts are designed to be ambient — felt rather than heard, providing directional context without interrupting normal activity. The primary modality is haptic (vibration); secondary modalities are audio (optional bone conduction) and visual (companion app).

No alert is automatic for anything except the standard alert classes. Emergency contact notification is always manually triggered.

---

## Haptic Alert System

### Directional Encoding

The haptic actuator that fires indicates the direction of the detected object relative to the wearer's body. The wearer does not need to interpret a compass direction — they feel the alert on the relevant body part.

**Directional mapping:**

| Approach direction | Primary haptic node | Secondary haptic node |
|---|---|---|
| Front | Pendant or belt | — |
| Front-left | Left bracelet | Pendant |
| Left | Left bracelet | Left anklet |
| Rear-left | Left anklet | Left bracelet |
| Rear | Both anklets simultaneously | — |
| Rear-right | Right anklet | Right bracelet |
| Right | Right bracelet | Right anklet |
| Front-right | Right bracelet | Pendant |
| Above | Pendant | Eyewear (if present) |
| Below (ground) | Left anklet + right anklet | — |

The "primary" node fires with full intensity; the "secondary" fires at half intensity to indicate that the alert source is between the two directions.

### Pattern Encoding

Haptic patterns encode alert class and urgency:

| Pattern | Duration | Meaning |
|---|---|---|
| Single pulse (300 ms) | Once | HumanApproaching slow (< 1 m/s relative) |
| Double pulse (100+100 ms, 200 ms gap) | Once | HumanApproaching fast (> 2 m/s relative) |
| Triple pulse (100+100+100 ms) | Once | VehicleNear |
| Sustained pulse (500 ms) | Once | GaitAnomaly — stumble precursor |
| Extended buzz (1000 ms) | Once, repeat after 3s if unacknowledged | FallDetected |
| Rapid pulses (50ms on/off × 5) | Once | FastObjectDetected (extreme velocity track) |
| Slow sweep (100 ms on, 100 ms off × 3) | Once | CalibrationPrompt |

---

## Body-Frame Stabilization and Alert Directionality

A critical aspect of haptic directionality is **body-frame stabilization**. When the wearer turns to face a different direction, stationary objects should not appear to "move" in the alert direction.

SENTINEL-WEAR uses **Translational stabilization** (default): the body frame co-translates with the wearer but does not co-rotate. This means:

- A person approaching from behind (bearing 180°) stays at bearing 180° even if the wearer turns around to look at them.
- The approaching person's track in the body frame only changes bearing when *the approaching person* changes direction, not when *the wearer* changes direction.

This makes the haptic direction intuitive — the alert fires on the node nearest to where the approaching person actually is relative to the wearer's current stance, not where they were when the track was first detected.

---

## Bone-Conduction Audio Alerts (Optional)

When a bone-conduction earpiece is paired:

| Alert | Audio cue |
|---|---|
| HumanApproaching | Tone with bearing-encoded frequency (higher pitch = more to the right) |
| VehicleNear | Distinct vehicle-class tone |
| FallDetected | Persistent tone until acknowledged |
| BatteryLow (any node) | Battery warning pattern |

Audio alerts are disabled by default and enabled in configuration.

---

## Companion App Display

The companion app shows a circular body-frame radar display:
- Center: the wearer
- Tracks shown as icons at relative bearing and estimated distance
- Icon color: class (human = blue, vehicle = orange, pet = green, unknown = white)
- Icon size: estimated distance (larger = closer)
- No imagery — this is a geometric display only

The display updates at the same rate as the belt controller's PentaTrack output.

---

## Emergency Contact

The emergency contact feature sends a location notification to a pre-configured contact. It is **always manually triggered**:

```bash
# From the companion app: Emergency button (long-press, 3 seconds)
# Or from CLI:
curl -X POST http://belt-node.local/api/alerts/emergency-contact
```

Fall detection does NOT automatically trigger emergency contact. The FallDetected alert fires a haptic pattern and is logged. If the wearer is incapacitated, they cannot trigger emergency contact — this is a known limitation. Users who need automatic emergency response should use a dedicated medical alert device in addition to SENTINEL-WEAR.
