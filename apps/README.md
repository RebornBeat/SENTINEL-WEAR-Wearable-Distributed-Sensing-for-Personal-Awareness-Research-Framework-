# SENTINEL-WEAR Companion App Architecture

**Status:** Specification
**Components:** Mobile app (iOS / Android), Desktop app (Windows / macOS / Linux)
**Server:** Embedded HTTP/WebSocket server on belt node

---

## 1. Overview

The SENTINEL-WEAR companion app connects to the belt node's embedded HTTP/WebSocket server over Wi-Fi or Bluetooth. It provides complete visibility into the distributed body-frame sensing mesh, full access to stored recordings, live sensor feeds, 360° panoramic views, legal export, and system configuration.

The app communicates exclusively with the user's own belt node. No data is transmitted to any third party. No telemetry. No external accounts required.

---

## 2. Mobile App (iOS / Android)

### Core Features

**Live Awareness:**
- Body-centric radar-style display showing all active nodes, detected entities as blobs with velocity vectors, PentaTrack prediction arcs, and alert zones as colored rings.
- Entity classification labels with confidence scores.
- Stumble prediction and gait event feed.

**Live Camera / Sensor Feeds:**
- On-demand live video from any camera-equipped node.
- 360° panoramic live view from curved pendant cameras (equirectangular or spherical display).
- Per-node audio stream.

**Recordings Library:**
- Browse all stored recordings: video, audio, sensor data, panoramas.
- Filter by node, trigger type, date, event classification.
- Full playback with timestamp scrubbing.
- Download individual recordings.

**Alert History:**
- Complete log of detection events, acoustic events, gait events, identification results.
- Each event links to associated recording if one was captured.

**Gait Analytics:**
- Stride history, cadence trends, stumble event timeline.
- Left/right asymmetry visualization.

**System Configuration:**
- Node management (add, remove, view status).
- Data handling policy (storage, streaming, retention).
- Alert sensitivity and directional thresholds.
- Operational mode (sensing-only, detection-triggered cameras, continuous recording, etc.).

**Remote Access:**
- When enabled by user, access belt node from outside the home network.
- Requires user-set bearer token and optionally TLS.

**Legal Export:**
- Export any recording with embedded integrity manifest.
- SHA-256 hash, device ID, firmware version, timestamp chain.
- One-tap share to Files / Documents.

### Connection Model

- **Primary:** Wi-Fi on same network as belt node.
- **Fallback:** BLE direct connection (limited to metadata and alerts — insufficient bandwidth for video).
- **Remote:** Encrypted tunnel via user-configured belt node remote endpoint.

---

## 3. Desktop App (Windows / macOS / Linux)

### Additional Features Beyond Mobile

**Multi-Node Recording Management:**
- Full timeline view across all nodes simultaneously.
- Tag, annotate, and search recordings.
- Bulk export with selection.

**3D World Model Viewer (Linux SoM belt variant):**
- Interactive 3D reconstruction of visited environments.
- Replay any past moment: scrub timeline to see 3D world state, entity trajectories, and events in spatial context.
- Useful for forensic review, security analysis, incident reconstruction.

**Body-Frame Analytics:**
- Visualize detection history in 3D body frame over time.
- Heatmaps of where detections occurred relative to the wearer.
- Replay detection trajectories.

**Gait Reports:**
- Long-term trends, anomaly history, comparison across days/weeks.

**Full Configuration Manager:**
- TOML editor with live validation.
- Node firmware update management.
- Calibration tools (manual position override, calibration replay).

**Legal Export Suite:**
- Export with full integrity chain including chain hash.
- Generate PDF report from integrity manifest suitable for submission.
- Batch export for date ranges.

**Backup and Archive:**
- Schedule automatic backup of recordings to external drives or network storage.
- Archive management with search.

---

## 4. Belt Node API Reference

### Authentication

All endpoints require a bearer token set by the user:
```
Authorization: Bearer <user_configured_token>
```

### REST API

| Method | Path | Description |
|---|---|---|
| GET | `/api/nodes` | All nodes with health, capabilities, battery percentage |
| GET | `/api/detections/recent` | Recent detections (last N, configurable) |
| GET | `/api/alerts` | Alert history (paginated, filterable by type/date) |
| GET | `/api/recordings` | List recordings (paginated, filterable) |
| GET | `/api/recordings/{id}` | Download recording file (binary) |
| GET | `/api/recordings/{id}/meta` | Integrity manifest JSON |
| DELETE | `/api/recordings/{id}` | Delete recording and associated manifest |
| GET | `/api/export/{id}` | Export recording with integrity chain (ZIP: file + manifest) |
| GET | `/api/nodes/{id}/stream` | Redirect to RTSP or WebSocket stream for this node |
| GET | `/api/nodes/{id}/panorama` | Redirect to 360° panorama stream for this node |
| GET | `/api/world-model` | Current world model state (entities, mode, SLAM available) |
| GET | `/api/world-model/replay/{timestamp_ms}` | World model state at past timestamp |
| GET | `/api/gait/history` | Gait event history (paginated) |
| GET | `/api/gait/metrics` | Current gait metrics window |
| GET | `/api/analytics/body-frame` | Detection heatmap data |
| GET | `/api/analytics/gait-trends` | Long-term gait trend data |
| GET | `/api/config` | Current full configuration |
| POST | `/api/config` | Update configuration (partial or full) |
| POST | `/api/policy/mode` | Change operational mode |
| POST | `/api/calibration/start` | Start walk-through calibration |
| POST | `/api/calibration/complete` | Complete calibration and apply |
| GET | `/api/health` | System health summary |

### WebSocket Streams

| Path | Protocol | Description |
|---|---|---|
| `/ws/events` | JSON | All live BAN events |
| `/ws/detections` | JSON | Live detection events only |
| `/ws/gait` | JSON | Live gait events |
| `/ws/alerts` | JSON | Live alert events |
| `/ws/world-model` | JSON | Live world model updates (5–20 Hz) |
| `/ws/node/{id}/sensor` | Binary | Raw sensor data stream (if configured) |

### Media Streaming

| Protocol | Port | Path | Description |
|---|---|---|---|
| RTSP | 9090 | `rtsp://belt:9090/{node_id}` | Standard RTSP per-node stream |
| WebSocket H.264 | 9091 | `/ws/media/{node_id}` | Browser/app-compatible H.264 stream |
| WebSocket 360° | 9092 | `/ws/panorama` | Equirectangular panorama stream |
| PCM Audio | 9093 | `/ws/audio/{node_id}` | Uncompressed audio stream |

---

## 5. Integrity Manifest Format

Every exported recording includes a JSON manifest:

```json
{
  "export_version": "1.0",
  "node_id": "pendant",
  "node_class": "Pendant360",
  "recording_start_utc": "2024-03-15T14:30:22.004Z",
  "recording_duration_s": 42.3,
  "format": "h264_mp4",
  "modalities": ["video", "360_video", "audio"],
  "file_sha256": "a3f4b2c9d8e1f7a2b5c8d3e6f9a0b1c2d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8",
  "firmware_version": "v1.0.0",
  "hardware_variant": "pendant_360_v1",
  "wearer_label": "user_configured_optional_label",
  "export_timestamp_utc": "2024-03-16T09:00:00.000Z",
  "chain_hash": "b7e8d1f3a2c4e5f6b7a8c9d0e1f2a3b4"
}
```

---

## 6. Security

- All API endpoints require authentication (user-set bearer token).
- TLS optional on local network; strongly recommended for remote access.
- Remote access token set by user; never transmitted in plaintext; stored hashed on belt node.
- No external telemetry. No third-party accounts. No cloud dependency.
- The app communicates exclusively with the user's own belt node.

---

## 7. Offline Operation

The belt node and all nodes operate fully without the companion app. The app is a viewer and manager — the sensing mesh runs independently. Recordings are stored on the belt node's SD card whether or not the app is connected. The app accesses them when reconnected.
