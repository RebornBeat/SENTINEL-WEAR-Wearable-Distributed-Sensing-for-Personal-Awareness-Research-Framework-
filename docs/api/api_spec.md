# SENTINEL-WEAR API Specification

**Base URL:** `http://belt-node.local` (on local network) or `http://<belt-ip>:8080`
**Protocol:** REST/JSON for configuration and queries; WebSocket for real-time streams; RTSP/WebSocket for media streams
**Authentication:** Bearer token required for all endpoints (configurable)

---

## Authentication

All API endpoints require authentication via bearer token set by the user:

```
Authorization: Bearer <user_configured_token>
```

Token is configured in `sentinel-wear.toml`:

```toml
[companion_app]
api_port = 8080
media_port = 9090
enable_remote_access = false
remote_access_token = "your_secure_token_here"
```

---

## Body-Frame Tracking

### `GET /api/body-frame/tracks`

All currently active tracks in the stabilized body frame.

**Response:**
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
      "timestamp_ns": 1706123456789,
      "prediction_centers": [
        {"time_ms": 100, "position": [4.5, 3.8, 0.0]},
        {"time_ms": 200, "position": [4.8, 3.2, 0.0]},
        {"time_ms": 300, "position": [5.1, 2.6, 0.0]}
      ],
      "anomaly_flags": []
    }
  ],
  "stabilization_mode": "Translational",
  "wearer_position_m": [0.0, 0.0, 1.7],
  "wearer_velocity_ms": [0.3, 0.8, 0.0],
  "timestamp_ns": 1706123456789
}
```

### `GET /api/body-frame/stream`

WebSocket endpoint. Emits `TrackUpdate` events in real time.

**Message format:**
```json
{
  "type": "TrackUpdate",
  "track_id": "trk_0007",
  "class": "HumanApproaching",
  "position": [4.2, 3.1, 0.9],
  "velocity": [0.8, 1.2, 0.0],
  "confidence": 0.91,
  "timestamp_ns": 1706123456789
}
```

### `GET /api/body-frame/entities`

Current tracked entities with full state.

**Response:**
```json
{
  "entities": [
    {
      "id": "ent_001",
      "track_id": "trk_0007",
      "class": "HumanApproaching",
      "position_body_frame": [4.2, 3.1, 0.9],
      "velocity_body_frame": [0.8, 1.2, 0.0],
      "range_m": 5.2,
      "bearing_rad": 3.14,
      "elevation_rad": -0.1,
      "confidence": 0.91,
      "first_seen_ns": 1706123400000,
      "last_seen_ns": 1706123456789,
      "prediction_horizon_ms": 500
    }
  ],
  "timestamp_ns": 1706123456789
}
```

### `GET /api/body-frame/replay/{timestamp_ms}`

Body-frame entity positions at a past timestamp (for playback/analysis).

**Path Parameters:**
- `timestamp_ms`: Unix timestamp in milliseconds

**Response:**
```json
{
  "timestamp_ms": 1706123400000,
  "entities": [
    {
      "id": "ent_001",
      "position_body_frame": [5.0, 4.2, 0.9],
      "velocity_body_frame": [0.8, 1.2, 0.0]
    }
  ],
  "wearer_position_m": [0.0, 0.0, 1.7],
  "wearer_velocity_ms": [0.3, 0.8, 0.0]
}
```

---

## Node Management

### `GET /api/nodes`

All registered nodes with health, battery, and capability status.

**Response:**
```json
{
  "nodes": [
    {
      "id": "pendant",
      "type": "PendantNode",
      "variant": "standard",
      "status": "Online",
      "battery_percent": 85,
      "battery_voltage_mv": 3950,
      "charging": false,
      "sensors": ["MmWaveRadar", "Acoustic", "Imu"],
      "camera": {
        "present": true,
        "enabled": false,
        "hardware_switch_state": "enabled"
      },
      "uwb": {
        "present": true,
        "enabled": false
      },
      "last_seen_ms": 42,
      "rssi_dbm": -58,
      "firmware_version": "v1.2.0",
      "calibration_confidence": 0.93
    },
    {
      "id": "belt",
      "type": "BeltNode",
      "variant": "linux_som",
      "status": "Online",
      "battery_percent": 72,
      "battery_voltage_mv": 3820,
      "charging": true,
      "sensors": ["MmWaveRadar", "Imu", "EnvSensor"],
      "external_interfaces": ["wifi", "cellular", "ble", "usb"],
      "uptime_hours": 12.5,
      "firmware_version": "v1.2.0",
      "thermal_c": 38.2
    }
  ],
  "total_nodes": 6,
  "online_nodes": 6,
  "timestamp_ns": 1706123456789
}
```

### `GET /api/nodes/{id}`

Detailed information for a single node.

**Path Parameters:**
- `id`: Node identifier (e.g., "pendant", "bracelet_left", "belt")

**Response:**
```json
{
  "id": "pendant",
  "type": "PendantNode",
  "variant": "standard",
  "status": "Online",
  "battery_percent": 85,
  "battery_voltage_mv": 3950,
  "charging": false,
  "estimated_runtime_hours": 18.5,
  "sensors": {
    "mmwave_radar": {
      "present": true,
      "status": "active",
      "last_update_ms": 12
    },
    "imu": {
      "present": true,
      "status": "active",
      "last_update_ms": 5
    },
    "acoustic": {
      "present": true,
      "status": "active",
      "microphone_count": 4,
      "last_update_ms": 20
    },
    "camera": {
      "present": true,
      "enabled": false,
      "hardware_switch_state": "enabled",
      "software_state": "disabled",
      "resolution": "1080p",
      "streaming": false
    }
  },
  "connectivity": {
    "ban_transport": "ble",
    "rssi_dbm": -58,
    "connection_interval_ms": 30,
    "last_ping_ms": 42
  },
  "calibration": {
    "position_m": [0.0, 0.15, 1.4],
    "orientation_quat": [0.998, 0.001, 0.002, 0.001],
    "confidence": 0.93,
    "last_calibrated": "2024-01-25T14:23:00Z"
  },
  "firmware_version": "v1.2.0",
  "hardware_revision": "v1.0"
}
```

### `GET /api/nodes/{id}/health`

Health metrics for a specific node.

**Response:**
```json
{
  "node_id": "pendant",
  "status": "Online",
  "health_score": 0.95,
  "issues": [],
  "sensors": [
    {"name": "mmwave_radar", "status": "OK", "last_update_ms": 12},
    {"name": "imu", "status": "OK", "last_update_ms": 5},
    {"name": "acoustic", "status": "OK", "last_update_ms": 20}
  ],
  "battery": {
    "percent": 85,
    "voltage_mv": 3950,
    "charging": false,
    "health_percent": 98,
    "cycle_count": 42
  },
  "connectivity": {
    "rssi_dbm": -58,
    "packet_loss_percent": 0.2,
    "latency_ms": 18
  },
  "thermal_c": 32.5,
  "uptime_hours": 12.3
}
```

### `GET /api/nodes/{id}/stream`

Open live media stream from a node. Redirects to the media streaming endpoint.

**Response:** Redirect to `rtsp://belt-node.local:9090/{node_id}` or WebSocket stream.

### `POST /api/nodes/{id}/restart`

Restart a specific node (sends restart command via BAN).

**Request:**
```json
{
  "delay_ms": 0
}
```

---

## Calibration

### `POST /api/calibration/neutral-pose/start`

Begin neutral-pose calibration. Wearer must stand in defined neutral pose.

**Response:**
```json
{
  "session_id": "cal_001",
  "phase": "neutral_pose",
  "status": "collecting",
  "duration_ms": 5000,
  "elapsed_ms": 0
}
```

### `POST /api/calibration/neutral-pose/complete`

Mark neutral-pose calibration as complete.

**Response:**
```json
{
  "session_id": "cal_001",
  "phase": "neutral_pose",
  "status": "complete",
  "per_node_confidence": {
    "pendant": 0.97,
    "bracelet_left": 0.95,
    "bracelet_right": 0.96,
    "anklet_left": 0.94,
    "anklet_right": 0.95,
    "belt": 1.0
  }
}
```

### `POST /api/calibration/walk/start`

Begin walk-through calibration.

**Response:**
```json
{
  "session_id": "cal_001",
  "phase": "walk_through",
  "status": "collecting",
  "min_steps_required": 10,
  "steps_recorded": 0
}
```

### `POST /api/calibration/walk/complete`

Mark walk-through calibration as complete.

**Response:**
```json
{
  "session_id": "cal_001",
  "phase": "walk_through",
  "status": "complete",
  "steps_recorded": 15,
  "per_node_confidence": {
    "pendant": 0.93,
    "bracelet_left": 0.91,
    "bracelet_right": 0.92,
    "anklet_left": 0.96,
    "anklet_right": 0.95,
    "belt": 1.0
  }
}
```

### `GET /api/calibration/status`

Current calibration state for all nodes.

**Response:**
```json
{
  "overall_status": "Calibrated",
  "overall_confidence": 0.94,
  "last_calibrated": "2024-01-25T14:30:00Z",
  "nodes": [
    {
      "id": "pendant",
      "status": "Calibrated",
      "confidence": 0.93,
      "position_m": [0.0, 0.15, 1.4],
      "orientation_quat": [0.998, 0.001, 0.002, 0.001]
    },
    {
      "id": "bracelet_left",
      "status": "Calibrated",
      "confidence": 0.91,
      "position_m": [-0.25, 0.1, 1.0],
      "orientation_quat": [0.995, 0.0, 0.0, 0.1]
    }
  ]
}
```

### `POST /api/calibration/recalibrate`

Force a complete recalibration sequence.

---

## Alerts

### `GET /api/alerts`

Alert history (paginated, filterable).

**Query Parameters:**
- `limit`: Number of results (default: 100)
- `offset`: Pagination offset
- `since`: Unix timestamp in milliseconds
- `class`: Filter by alert class
- `min_priority`: Filter by minimum priority ("info", "warning", "critical")

**Response:**
```json
{
  "alerts": [
    {
      "id": "alert_001",
      "class": "HumanApproaching",
      "priority": "warning",
      "bearing_rad": 3.14,
      "range_m": 4.2,
      "velocity_ms": 1.5,
      "confidence": 0.88,
      "node_id": "pendant",
      "timestamp_ns": 1706123456789,
      "acknowledged": false,
      "recording_id": null
    },
    {
      "id": "alert_002",
      "class": "GaitAnomaly",
      "priority": "warning",
      "anomaly_type": "stumble_precursor",
      "confidence": 0.82,
      "node_id": "anklet_left",
      "timestamp_ns": 1706123420000,
      "acknowledged": true,
      "recording_id": "rec_0042"
    }
  ],
  "total": 42,
  "offset": 0,
  "limit": 100
}
```

### `GET /api/alerts/recent`

Recent alert history (last 100 events, convenience endpoint).

### `GET /api/alerts/stream`

WebSocket endpoint. Real-time alert stream.

**Message format:**
```json
{
  "type": "AlertEvent",
  "id": "alert_001",
  "class": "HumanApproaching",
  "priority": "warning",
  "bearing_rad": 3.14,
  "range_m": 4.2,
  "velocity_ms": 1.5,
  "confidence": 0.88,
  "node_id": "pendant",
  "timestamp_ns": 1706123456789,
  "haptic_triggered": true,
  "recording_started": false
}
```

### `POST /api/alerts/{id}/acknowledge`

Acknowledge an alert.

### `POST /api/alerts/emergency-contact`

**Manual trigger only.** Sends location notification to configured emergency contact.

**Request:**
```json
{
  "message": "Manual emergency alert triggered by user",
  "include_location": true,
  "include_recent_clip": true
}
```

**Response:**
```json
{
  "status": "sent",
  "contact": "+15551234567",
  "message_id": "emsg_001",
  "location_included": true,
  "clip_included": true,
  "clip_duration_s": 10
}
```

---

## Connectivity

### `GET /api/connectivity`

Current connectivity status for all interfaces.

**Response:**
```json
{
  "wifi": {
    "connected": true,
    "ssid": "HomeNetwork",
    "rssi_dbm": -52,
    "ip_address": "192.168.1.100",
    "band": "5ghz",
    "channel": 36
  },
  "cellular": {
    "enabled": true,
    "connected": true,
    "carrier": "Carrier Name",
    "technology": "LTE",
    "rssi_dbm": -78,
    "signal_quality": "good",
    "data_used_mb": 42.5,
    "sim_type": "nano_sim"
  },
  "bluetooth": {
    "enabled": true,
    "advertising": true,
    "paired_devices": 1,
    "connection_mode": "ble_direct_available"
  },
  "ban": {
    "transport": "ble",
    "nodes_connected": 6,
    "total_nodes": 6,
    "connection_interval_ms": 30
  },
  "usb": {
    "connected": false
  }
}
```

### `GET /api/connectivity/wifi`

WiFi-specific status and settings.

**Response:**
```json
{
  "enabled": true,
  "connected": true,
  "ssid": "HomeNetwork",
  "rssi_dbm": -52,
  "ip_address": "192.168.1.100",
  "netmask": "255.255.255.0",
  "gateway": "192.168.1.1",
  "band": "5ghz",
  "channel": 36,
  "prefer_5ghz": true,
  "mac_address": "AA:BB:CC:DD:EE:FF"
}
```

### `POST /api/connectivity/wifi/connect`

Connect to a WiFi network.

**Request:**
```json
{
  "ssid": "NewNetwork",
  "password": "network_password",
  "prefer_5ghz": true
}
```

### `POST /api/connectivity/wifi/disconnect`

Disconnect from current WiFi network.

### `POST /api/connectivity/wifi/scan`

Scan for available WiFi networks.

**Response:**
```json
{
  "networks": [
    {
      "ssid": "Network1",
      "rssi_dbm": -45,
      "band": "5ghz",
      "channel": 36,
      "security": "wpa2_psk",
      "open": false
    }
  ]
}
```

### `GET /api/connectivity/cellular`

Cellular status and signal information.

**Response:**
```json
{
  "enabled": true,
  "connected": true,
  "module": "Quectel EC25",
  "carrier": "Carrier Name",
  "technology": "LTE",
  "rssi_dbm": -78,
  "rsrp_dbm": -95,
  "rsrq_db": -12,
  "signal_quality": "good",
  "sim_type": "nano_sim",
  "sim_present": true,
  "imei": "123456789012345",
  "imsi": "123456789012345",
  "data_used_mb": 42.5,
  "data_limit_mb": 1000,
  "roaming": false,
  "apn": "carrier.apn"
}
```

### `POST /api/connectivity/cellular/toggle`

Enable or disable cellular module.

**Request:**
```json
{
  "enabled": true
}
```

### `POST /api/connectivity/cellular/configure`

Configure cellular settings.

**Request:**
```json
{
  "apn": "new.carrier.apn",
  "fallback_to_wifi": true,
  "always_on_cellular": false,
  "alert_via_cellular": true,
  "stream_via_cellular": false
}
```

### `GET /api/connectivity/cellular/data-usage`

Data usage statistics.

**Response:**
```json
{
  "current_period": {
    "start_date": "2024-01-01",
    "end_date": "2024-01-31",
    "used_mb": 42.5,
    "limit_mb": 1000,
    "percent_used": 4.25
  },
  "daily": [
    {"date": "2024-01-25", "used_mb": 1.2},
    {"date": "2024-01-24", "used_mb": 0.8}
  ]
}
```

### `GET /api/connectivity/ban`

BAN network status.

**Response:**
```json
{
  "transport": "ble",
  "connection_interval_ms": 30,
  "uwb_enabled": false,
  "nodes": [
    {
      "id": "pendant",
      "connected": true,
      "rssi_dbm": -58,
      "last_ping_ms": 42
    },
    {
      "id": "bracelet_left",
      "connected": true,
      "rssi_dbm": -62,
      "last_ping_ms": 38
    }
  ],
  "scheduler_mode": "dynamic",
  "qos_enabled": true,
  "antenna_diversity": {
    "enabled": true,
    "active_antenna": "right"
  }
}
```

---

## 360° Panorama and Media

### `GET /api/nodes/pendant_360/panorama/latest`

Latest stitched panorama frame from the 360° pendant (JPEG equirectangular).

**Response:** Binary JPEG image.

**Headers:**
- `Content-Type: image/jpeg`
- `X-Timestamp-Ns: 1706123456789`
- `X-Resolution: 3840x1920`
- `X-Stitch-Quality: 0.92`

### `GET /ws/panorama`

WebSocket endpoint for live 360° panorama stream.

**Protocol:** Binary frames (JPEG or H.264 NAL units, negotiated at connection).

**Connection Parameters:**
- `resolution`: "qvga" | "vga" | "720p" | "1080p" | "2k" | "4k"
- `quality`: "low" | "medium" | "high"
- `fps`: 1-30

**Example:**
```
ws://belt-node.local:9092/panorama?resolution=1080p&quality=medium&fps=15
```

### `GET /api/nodes/{id}/panorama/calibration`

Camera calibration status for 360° pendant.

**Response:**
```json
{
  "node_id": "pendant_360",
  "camera_count": 8,
  "calibration_status": "calibrated",
  "calibration_date": "2024-01-25T14:30:00Z",
  "rms_reprojection_error_px": 0.62,
  "per_camera": [
    {
      "index": 0,
      "angle_deg": 0,
      "intrinsics_calibrated": true,
      "extrinsics_calibrated": true
    }
  ],
  "stitching_quality": 0.92
}
```

### `POST /api/nodes/pendant_360/panorama/calibrate`

Start 360° camera calibration procedure.

**Request:**
```json
{
  "type": "full",
  "positions_required": 8
}
```

### `GET /api/media/ports`

Available media streaming ports and protocols.

**Response:**
```json
{
  "rtsp_port": 9090,
  "websocket_h264_port": 9091,
  "websocket_panorama_port": 9092,
  "pcm_audio_port": 9093,
  "protocols": ["rtsp", "websocket_h264", "websocket_mjpeg", "pcm_audio"]
}
```

---

## Gait and Body Analytics

### `GET /api/gait/live`

Current gait metrics window.

**Response:**
```json
{
  "current_phase": "stance_left",
  "cadence_steps_per_min": 112,
  "stride_length_m": 0.72,
  "stride_variability_percent": 3.2,
  "left_right_asymmetry_percent": 1.8,
  "impact_g": 5.2,
  "walking_speed_ms": 1.3,
  "stability_score": 0.91,
  "anomaly_detected": false,
  "last_step_ns": 1706123456789
}
```

### `GET /api/gait/history`

Gait event history (paginated).

**Query Parameters:**
- `limit`: Number of results
- `since`: Unix timestamp in milliseconds
- `event_type`: Filter by event type

**Response:**
```json
{
  "events": [
    {
      "id": "gait_001",
      "type": "step",
      "phase": "heel_strike",
      "foot": "left",
      "impact_g": 5.2,
      "timestamp_ns": 1706123456789
    },
    {
      "id": "gait_002",
      "type": "stumble_precursor",
      "severity": "minor",
      "confidence": 0.78,
      "timestamp_ns": 1706123420000
    }
  ],
  "total": 1250,
  "offset": 0,
  "limit": 100
}
```

### `GET /api/gait/stumble-events`

Stumble precursor and fall events specifically.

**Response:**
```json
{
  "events": [
    {
      "id": "stumble_001",
      "type": "stumble_precursor",
      "severity": "minor",
      "confidence": 0.78,
      "recovering": true,
      "impact_g": 8.2,
      "node_id": "anklet_left",
      "timestamp_ns": 1706123420000,
      "acknowledged": false
    }
  ]
}
```

### `GET /api/gait/metrics/summary`

Long-term gait trend analysis.

**Query Parameters:**
- `period`: "hour" | "day" | "week" | "month"

**Response:**
```json
{
  "period": "week",
  "total_steps": 42350,
  "average_cadence": 108,
  "average_stride_length_m": 0.71,
  "average_walking_speed_ms": 1.28,
  "asymmetry_trend": "stable",
  "stumble_count": 2,
  "fall_count": 0,
  "stability_trend": "improving"
}
```

### `GET /api/analytics/body-frame`

Detection heatmap data in body-frame coordinates.

**Query Parameters:**
- `period`: Analysis period
- `resolution_deg`: Angular resolution for heatmap bins

**Response:**
```json
{
  "heatmap": {
    "bins": [
      {"bearing_deg": 0, "elevation_deg": 0, "detection_count": 42},
      {"bearing_deg": 45, "elevation_deg": 0, "detection_count": 38}
    ],
    "resolution_deg": 15,
    "period_start_ns": 1706000000000,
    "period_end_ns": 1706123456789
  },
  "most_common_approach_direction_deg": 0,
  "total_detections": 1250
}
```

---

## SLAM World Model

### `GET /api/world-model`

Current world model state and availability.

**Response:**
```json
{
  "mode_a_sparse": {
    "available": true,
    "active": true,
    "tracked_entities": 3,
    "update_rate_hz": 20
  },
  "mode_b_dense_slam": {
    "available": true,
    "active": true,
    "map_size_mb": 42.5,
    "keyframe_count": 1250,
    "loop_closures": 15,
    "drift_estimate_m": 0.08,
    "update_rate_hz": 5
  },
  "hardware": {
    "belt_variant": "linux_som",
    "compute_available": true,
    "storage_available_mb": 15000
  }
}
```

### `GET /api/world-model/3d`

Current 3D SLAM mesh (if available).

**Response:** Binary PLY or OBJ mesh file.

**Headers:**
- `Content-Type: model/ply`
- `X-Vertex-Count: 125000`
- `X-Mesh-Size-Mb: 5.2`

### `GET /api/world-model/replay/{timestamp_ms}`

World model state at a past timestamp.

**Response:**
```json
{
  "timestamp_ms": 1706123400000,
  "entities": [
    {
      "id": "ent_001",
      "position_world_m": [2.1, 3.4, 0.9],
      "velocity_ms": [0.8, 1.2, 0.0],
      "class": "HumanApproaching"
    }
  ],
  "wearer_position_world_m": [1.0, 1.0, 0.0],
  "keyframe_ids": ["kf_001", "kf_002"]
}
```

### `POST /api/world-model/segments/delete`

Delete a specific location segment from the SLAM map.

**Request:**
```json
{
  "segment_id": "office_building",
  "confirm": true
}
```

---

## Identification (Opt-In Only)

### `GET /api/identification/status`

Identification sensor status.

**Response:**
```json
{
  "nodes_with_identification": [
    {
      "node_id": "pendant",
      "sensor_present": true,
      "hardware_switch_state": "enabled",
      "software_state": "disabled",
      "last_identification_ns": null,
      "mode": "metadata_only"
    }
  ],
  "total_identification_nodes": 1,
  "active_nodes": 0
}
```

### `POST /api/identification/enable`

Enable identification on a specific node.

**Request:**
```json
{
  "node_id": "pendant"
}
```

**Response:**
```json
{
  "node_id": "pendant",
  "status": "enabled",
  "mode": "metadata_only",
  "hardware_switch_state": "enabled"
}
```

### `POST /api/identification/disable`

Disable identification on a specific node.

### `GET /api/identification/events`

Recent identification events.

**Response:**
```json
{
  "events": [
    {
      "id": "id_001",
      "node_id": "pendant",
      "classification": "KnownResident",
      "confidence": 0.94,
      "timestamp_ns": 1706123456789,
      "recording_available": true,
      "recording_id": "rec_0042"
    }
  ]
}
```

---

## Extreme Velocity Detection

### `GET /api/extreme-velocity/status`

Extreme velocity detection system status.

**Response:**
```json
{
  "enabled": false,
  "mode": "disabled",
  "nodes_with_doppler": ["pendant"],
  "nodes_with_event_camera": ["pendant", "eyewear"],
  "doppler_config": {
    "update_rate_hz": 10000,
    "velocity_range_ms": 300,
    "range_resolution_m": 0.5
  },
  "latency_target_ms": 5,
  "qos_priority": "critical"
}
```

### `POST /api/extreme-velocity/enable`

Enable extreme-velocity sensing mode.

**Request:**
```json
{
  "mode": "production",
  "min_velocity_ms": 50,
  "max_detection_distance_m": 10,
  "alert_on_detection": true
}
```

### `POST /api/extreme-velocity/disable`

Disable extreme-velocity sensing mode.

### `GET /api/extreme-velocity/events`

Recent fast-object detection events.

**Response:**
```json
{
  "events": [
    {
      "id": "ev_001",
      "timestamp_ns": 1706123456789,
      "velocity_ms": 320,
      "bearing_rad": 1.57,
      "elevation_rad": 0.0,
      "detection_range_m": 8.2,
      "time_to_impact_ms": 25.6,
      "confidence": 0.91,
      "source_nodes": ["pendant", "eyewear"],
      "classification": "UnknownFastObject",
      "alert_triggered": true
    }
  ],
  "total": 5
}
```

### `GET /api/extreme-velocity/config`

Current extreme velocity configuration.

**Response:**
```json
{
  "enabled": false,
  "mode": "disabled",
  "doppler_update_rate_hz": 10000,
  "event_camera_always_on": false,
  "min_velocity_ms": 50,
  "max_detection_distance_m": 10,
  "alert_on_detection": true,
  "processing_priority": "realtime",
  "qos_class": "critical",
  "reserved_slot_ms": [25, 28]
}
```

### `POST /api/extreme-velocity/config`

Update extreme velocity configuration.

**Request:**
```json
{
  "enabled": true,
  "mode": "production",
  "doppler_update_rate_hz": 10000,
  "min_velocity_ms": 100,
  "max_detection_distance_m": 8,
  "alert_on_detection": true
}
```

---

## Recordings

### `GET /api/recordings`

List stored recordings (paginated, filterable).

**Query Parameters:**
- `limit`: Number of results
- `offset`: Pagination offset
- `node_id`: Filter by source node
- `since`: Unix timestamp in milliseconds
- `trigger`: Filter by trigger type ("detection", "alert", "manual", "continuous")
- `modality`: Filter by data type ("video", "audio", "point_cloud", "panorama")

**Response:**
```json
{
  "recordings": [
    {
      "id": "rec_0042",
      "node_id": "pendant",
      "start_timestamp_ns": 1706123400000,
      "duration_ms": 12800,
      "size_bytes": 45231088,
      "format": "h264_mp4",
      "modalities": ["video", "audio"],
      "trigger": "on_detection",
      "trigger_event_id": "alert_001",
      "integrity_sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
      "available": true
    }
  ],
  "total": 125,
  "storage_used_mb": 5420,
  "storage_limit_mb": 32000
}
```

### `GET /api/recordings/{id}`

Download a recording file.

**Response:** Binary file stream.

**Headers:**
- `Content-Type: video/mp4` (or appropriate MIME type)
- `Content-Length: 45231088`
- `X-Recording-Id: rec_0042`
- `X-Integrity-Sha256: a3f4b2c9...`

### `GET /api/recordings/{id}/meta`

Recording metadata and integrity manifest.

**Response:**
```json
{
  "id": "rec_0042",
  "node_id": "pendant",
  "node_type": "PendantNode",
  "start_timestamp_ns": 1706123400000,
  "duration_ms": 12800,
  "size_bytes": 45231088,
  "format": "h264_mp4",
  "modalities": ["video", "audio"],
  "trigger": "on_detection",
  "trigger_event_id": "alert_001",
  "integrity": {
    "sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
    "firmware_version": "v1.2.0",
    "hardware_revision": "v1.0",
    "chain_hash": "b7e8d1f3a2c4e5f6b7a8c9d0e1f2a3b4"
  },
  "wearer_label": "user_configured_label",
  "body_frame_position": [0.0, 0.15, 1.4],
  "location_segment": "home"
}
```

### `DELETE /api/recordings/{id}`

Delete a recording.

**Response:**
```json
{
  "status": "deleted",
  "id": "rec_0042",
  "freed_mb": 43.1
}
```

### `GET /api/export/{id}`

Export recording with integrity manifest (ZIP archive).

**Response:** ZIP file containing:
- Recording file
- `manifest.json` with integrity data

### `POST /api/export/bulk`

Bulk export for a date range.

**Request:**
```json
{
  "start_timestamp_ns": 1706000000000,
  "end_timestamp_ns": 1706123456789,
  "node_ids": ["pendant", "bracelet_left"],
  "include_manifest": true
}
```

### `GET /api/export/{id}/report`

Generate formatted PDF integrity report suitable for legal/insurance submission.

**Response:** PDF document.

---

## Configuration

### `GET /api/config`

Current full system configuration.

**Response:**
```json
{
  "nodes": {
    "pendant": {
      "sensors": ["MmWaveRadar", "Acoustic", "Imu"],
      "variant": "standard",
      "enabled": true
    },
    "belt": {
      "sensors": ["MmWaveRadar", "Imu", "EnvSensor"],
      "variant": "linux_som"
    }
  },
  "connectivity": {
    "wifi_enabled": true,
    "wifi_prefer_5ghz": true,
    "cellular": {
      "enabled": false,
      "sim_type": "nano_sim",
      "apn": "",
      "fallback_to_wifi": true
    }
  },
  "ban": {
    "primary": "ble",
    "ble_connection_interval_ms": 30,
    "uwb_enabled": false,
    "uwb_role": "timing_and_ranging",
    "scheduling_mode": "dynamic",
    "antenna_diversity": {
      "enabled": true,
      "antenna_count": 2
    }
  },
  "data": {
    "store_raw_video": true,
    "store_raw_audio": false,
    "storage_target": "sd_card",
    "retention_days": 30,
    "recording_trigger": "on_detection"
  },
  "extreme_velocity": {
    "enabled": false,
    "mode": "disabled"
  }
}
```

### `POST /api/config`

Update system configuration (partial or full).

**Request:**
```json
{
  "ban": {
    "ble_connection_interval_ms": 15
  }
}
```

### `POST /api/config/reset`

Reset configuration to defaults.

### `GET /api/config/schema`

JSON schema for configuration validation.

---

## Policy Modes

### `GET /api/policy`

Current policy mode and rules.

**Response:**
```json
{
  "current_mode": "standard",
  "available_modes": ["minimal", "standard", "security", "stealth"],
  "rules": {
    "alert_on_human_approach": true,
    "alert_on_vehicle_near": true,
    "alert_on_gait_anomaly": true,
    "camera_trigger": "on_detection",
    "recording_mode": "on_alert"
  }
}
```

### `POST /api/policy/mode`

Change operational policy mode.

**Request:**
```json
{
  "mode": "security",
  "custom_rules": {
    "alert_radius_m": 10,
    "camera_trigger": "on_alert"
  }
}
```

---

## Thermal and Power Management

### `GET /api/thermal`

Thermal status for all nodes.

**Response:**
```json
{
  "nodes": [
    {
      "node_id": "belt",
      "temperature_c": 38.2,
      "status": "normal",
      "throttled": false,
      "throttle_reason": null
    },
    {
      "node_id": "pendant_360",
      "temperature_c": 42.5,
      "status": "warm",
      "throttled": true,
      "throttle_reason": "camera_fps_reduced"
    }
  ],
  "ambient_c": 25.0
}
```

### `GET /api/power`

Power status and battery information.

**Response:**
```json
{
  "nodes": [
    {
      "node_id": "belt",
      "battery_percent": 72,
      "battery_voltage_mv": 3820,
      "charging": true,
      "estimated_runtime_hours": 14.5,
      "power_mode": "standard"
    }
  ],
  "total_capacity_mah": 5000,
  "total_charge_percent": 72
}
```

### `POST /api/power/mode`

Set power management mode.

**Request:**
```json
{
  "mode": "power_saving"
}
```

---

## WebSocket Endpoints Summary

| Path | Description | Message Format |
|------|-------------|----------------|
| `/ws/events` | All live BAN and system events | JSON |
| `/ws/detections` | Live detection events only | JSON |
| `/ws/gait` | Live gait events | JSON |
| `/ws/alerts` | Live alert events | JSON |
| `/ws/world-model` | World model updates | JSON |
| `/ws/panorama` | Live 360° panorama stream | Binary (JPEG/H.264) |
| `/ws/node/{id}/sensor` | Raw sensor data stream | Binary |

---

## Media Streaming Ports

| Protocol | Port | Description |
|----------|------|-------------|
| RTSP | 9090 | Per-node video stream (RTSP) |
| WebSocket H.264 | 9091 | Browser/app compatible H.264 stream |
| WebSocket Panorama | 9092 | 360° equirectangular panorama stream |
| PCM Audio | 9093 | Per-node raw audio stream |

---

## Error Responses

All endpoints return standard HTTP error codes:

| Code | Description |
|------|-------------|
| 400 | Bad Request — Invalid parameters |
| 401 | Unauthorized — Missing or invalid token |
| 403 | Forbidden — Operation not permitted |
| 404 | Not Found — Resource doesn't exist |
| 409 | Conflict — Resource state conflict |
| 429 | Too Many Requests — Rate limited |
| 500 | Internal Server Error |

**Error response format:**
```json
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "Connection interval must be between 7.5 and 1000 ms",
    "field": "ban.ble_connection_interval_ms"
  }
}
```

---

## Rate Limiting

| Endpoint Category | Limit |
|-------------------|-------|
| Configuration (POST) | 10 requests/minute |
| Recordings (GET) | 60 requests/minute |
| Recordings (DELETE) | 30 requests/minute |
| All other GET | 120 requests/minute |
| WebSocket connections | 5 concurrent |

---

**End of API Specification**
