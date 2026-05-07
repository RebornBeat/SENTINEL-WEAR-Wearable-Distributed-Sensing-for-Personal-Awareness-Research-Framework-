# SENTINEL-WEAR API Specification

**Base URL:** `http://belt-node.local` (on local network) or BLE companion app pairing
**Protocol:** REST/JSON for configuration; WebSocket for real-time streams; BAN protocol for node-to-belt communication (not user-facing)

---

## Body Frame

### `GET /api/body-frame/tracks`

All currently active tracks in the stabilized body frame.

```json
{
  "tracks": [
    {
      "id": "trk_0007",
      "class": "HumanApproaching",
      "bearing_rad": 3.14,
      "elevation_rad": -0.1,
      "range_m": 4.2,
      "relative_velocity_ms": 1.5,
      "confidence": 0.88,
      "timestamp_ns": 1706123456789
    }
  ],
  "stabilization_mode": "Translational"
}
```

### `GET /api/body-frame/stream`

WebSocket. Emits `TrackUpdate` events in real time.

---

## Node Management

### `GET /api/nodes`

All registered nodes with health and battery status.

### `GET /api/nodes/{id}/health`

Detailed health for one node.

---

## Calibration

### `POST /api/calibration/neutral-pose/start`

Begin neutral-pose calibration. Wearer must stand in defined neutral pose.

### `POST /api/calibration/neutral-pose/complete`

Mark neutral-pose complete.

### `POST /api/calibration/walk/start`

Begin walk-through calibration.

### `POST /api/calibration/walk/complete`

Mark walk-through complete.

### `GET /api/calibration/status`

Per-node calibration state and confidence.

---

## Alerts

### `GET /api/alerts/recent`

Recent alert history (last 100 events).

### `GET /api/alerts/stream`

WebSocket. Real-time alert stream.

### `POST /api/alerts/emergency-contact`

**Manual trigger only.** Sends location notification to configured emergency contact.

---

## Identification (Opt-In Only)

### `GET /api/identification/status`

Identification sensor status (enabled/disabled, kill-switch state).

### `POST /api/identification/enable`

Enable identification sensor (requires kill-switch to be in enabled position).

### `POST /api/identification/disable`

Disable identification sensor.

---

## Extreme Velocity Track (Research Mode)

### `POST /api/extreme-velocity/enable`

Enable extreme-velocity sensing mode (activates CW Doppler + event camera pipeline).

### `GET /api/extreme-velocity/events`

Recent fast-object detection events with velocity, bearing, and timing data.
