# SENTINEL-WEAR Companion App Architecture

**Status:** Specification
**Components:** Mobile app (iOS / Android), Desktop app (Windows / macOS / Linux)
**Server:** Embedded HTTP/WebSocket server on belt node

---

## 1. Overview

The SENTINEL-WEAR companion app connects to the belt node's embedded HTTP/WebSocket server over Wi-Fi, Bluetooth, or cellular (when remote access is enabled). It provides complete visibility into the distributed body-frame sensing mesh, full access to stored recordings, live sensor feeds, 360° panoramic views, extreme velocity detection alerts, legal export, and system configuration.

**Critical architectural principle:** The app communicates exclusively with the user's own belt node. The belt node is the **sole external gateway** for the entire wearable sensing mesh. No pendant, bracelet, anklet, or eyewear node connects to the app directly — all data routes through the belt node.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     COMPANION APP CONNECTIVITY MODEL                         │
│                                                                              │
│  Pendant ──┐                                                                 │
│  Bracelets─┤                                                                 │
│  Anklets───┼── BAN (BLE 5.x / UWB) ──► Belt Node ──► WiFi ──► Companion App │
│  Eyewear───┘                               │                                 │
│                                            │                                 │
│                                            ├──► Cellular (LTE/5G)           │
│                                            │         │                       │
│                                            │         ▼                       │
│                                            │    Remote Companion App         │
│                                            │                                 │
│                                            └──► BLE Direct (Fallback)       │
│                                                      │                       │
│                                                      ▼                       │
│                                              Local Companion App            │
│                                              (Limited: alerts, config only)  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**No data is transmitted to any third party. No telemetry. No external accounts required.**

---

## 2. Connection Modes

### Primary: Wi-Fi (Local Network)

**Bandwidth:** 50-300+ Mbps  
**Latency:** 5-20 ms  
**Capabilities:** All features — live streaming, 360° panorama, SLAM data, full recordings access

**Discovery:** App automatically discovers belt node via mDNS (`sentinel-wear.local`) when on the same network.

**Manual connection:** If mDNS unavailable, user enters belt node IP address directly.

### Secondary: Cellular (Remote Access)

**Enabled when:** User configures cellular module on belt node and enables remote access.

**Bandwidth:**  
- LTE Cat 1: 5-10 Mbps — alerts, metadata, compressed clips  
- LTE Cat 4: 50-150 Mbps — moderate streaming  
- 5G: 100-1000+ Mbps — full 360° live streaming

**Requirements:**
- User-set remote access token
- TLS encryption required
- Belt node has active cellular connection

**Features available:**
- Full remote control and configuration
- Live streaming (bandwidth-dependent quality)
- Alert push notifications
- Recordings access and download

### Fallback: Bluetooth (BLE 5.x Direct)

**Bandwidth:** 500 Kbps - 2 Mbps (practical ~500 Kbps)  
**Latency:** 20-100 ms  
**Capabilities:** Limited — alerts, metadata, configuration, node status

**Insufficient for:**
- Live video streaming
- 360° panorama
- Recording playback
- SLAM data

**Use when:**
- No Wi-Fi available
- No cellular configured
- Emergency minimal access

### Connection Priority

```
1. Wi-Fi (local network) → Always preferred when available
2. Cellular (remote) → Used when Wi-Fi unavailable and configured
3. BLE direct → Fallback only, limited functionality
```

**Configuration:**
```toml
[companion_app]
connection_priority = ["wifi", "cellular", "ble"]
ble_fallback_enabled = true
auto_reconnect = true
connection_timeout_ms = 5000
```

---

## 3. Mobile App (iOS / Android)

### Core Features

#### Live Awareness Display

**Body-centric radar-style view:**
- Wearer at center of display
- Active nodes shown as icons at their body positions (pendant, bracelets, anklets)
- Detected entities displayed as blobs with:
  - Position relative to wearer (distance and bearing)
  - Velocity vectors showing direction and speed
  - PentaTrack prediction arcs (where entity is heading)
  - Classification label (human, vehicle, animal, unknown)
  - Confidence score (opacity/size encoding)
- Alert zones shown as colored rings:
  - Yellow: detection zone
  - Orange: approaching zone
  - Red: alert zone (immediate attention)

**Real-time updates:**
- 5-20 Hz refresh rate (configurable)
- Latency: < 50 ms on Wi-Fi, < 200 ms on cellular
- Motion trails showing recent trajectory

**Gesture controls:**
- Pinch to zoom (adjust range scale)
- Drag to rotate view
- Tap on entity for details
- Long-press on zone to edit

#### Extreme Velocity Alerts

**Special alert mode for fast-moving objects:**

When extreme velocity detection is enabled and a fast object is detected:
- Immediate full-screen alert overlay
- Visual: Red pulse animation, bearing indicator
- Audio: Distinct urgent tone
- Haptic: Pattern transmitted to wearable nodes
- Information displayed:
  - Detected velocity (m/s)
  - Estimated time to closest approach
  - Bearing from wearer
  - Detection confidence

**Alert dismissal:**
- Tap to acknowledge
- Auto-dismiss after configurable timeout (default: 10 seconds)
- Logged in alert history with full details

#### Live Camera / Sensor Feeds

**On-demand live video:**
- Tap any camera-equipped node to view live feed
- Supported nodes: pendant (standard or 360°), bracelets (if camera configured), eyewear
- Resolution/quality adaptive based on connection:
  - Wi-Fi: Full resolution
  - Cellular LTE Cat 4: 720p
  - Cellular LTE Cat 1: 480p
  - BLE: Not available

**360° panoramic live view (pendant 360° variant):**
- Equirectangular view: Swipe left-right to pan through full 360°
- Spherical view (VR-compatible): Tilt/rotate device to look around
- VR headset mode: Compatible with mobile VR headsets
- Bearing indicators: Shows which direction wearer is facing relative to camera view
- Entity overlays: Detections shown as bounding boxes in the panorama

**Per-node audio stream:**
- Tap to listen to audio from any node with microphone
- Real-time audio (PCM stream)
- Optional recording during live listen

**Stream quality selection:**
```
Settings → Streaming Quality:
- High (full resolution, higher bandwidth)
- Medium (balanced)
- Low (reduced bandwidth, lower resolution)
- Adaptive (automatic based on connection)
```

#### Recordings Library

**Browse recordings:**
- Timeline view: Scrub through all recordings chronologically
- Grid view: Thumbnail preview of each recording
- List view: Detailed metadata for each recording

**Filter options:**
- By node: pendant, bracelet_left, bracelet_right, anklet_left, anklet_right, eyewear
- By trigger: on_detection, on_alert, manual, continuous
- By date range: Today, This week, This month, Custom range
- By event type: HumanApproaching, VehicleNear, FallDetected, GaitAnomaly, FastObjectDetected
- By modality: video, audio, panorama, sensor_data

**Playback controls:**
- Play/pause
- Scrub timeline
- Playback speed: 0.25x, 0.5x, 1x, 2x, 4x
- Frame-by-frame (for video recordings)
- Loop section (select start/end points)

**Download:**
- Download to device for offline access
- Select quality for download
- Batch download selected recordings

#### Alert History

**Complete log of all detection events:**
- Chronological list with timestamps
- Event classification and confidence
- Associated node
- Location context (body position: front, left, right, rear)
- Link to associated recording (if captured)

**Alert detail view:**
- Full event metadata
- Map view showing detection position relative to wearer
- Playback of associated recording (one tap)
- Export event with recording

**Filter and search:**
- By alert type
- By date range
- By node
- Search by classification keyword

#### Gait Analytics

**Real-time gait display:**
- Current cadence (steps/minute)
- Step regularity indicator
- Asymmetry meter (left vs. right)
- Impact force (g) from last heel-strike

**Historical gait data:**
- Cadence trend graph (hourly, daily, weekly views)
- Asymmetry trend over time
- Stumble event timeline with severity markers
- Impact force history

**Anomaly highlighting:**
- Stumble precursor events highlighted
- Pattern changes flagged
- Correlation with time of day / activity level

#### System Configuration

**Node management:**
- List all paired nodes with status
- Battery level per node
- Signal quality per node (BLE/UWB)
- Add new node (pairing mode)
- Remove node
- Node details: firmware version, hardware variant, calibration status

**Data handling policy:**
- Store raw video: On/Off
- Store raw audio: On/Off
- Store raw sensor data: On/Off
- Storage target: SD card / Internal / Network
- Retention period: Days (0 = forever)
- Continuous recording: On/Off
- Recording trigger: Always / On Detection / On Alert / Manual
- Maximum storage: MB (0 = unlimited)

**Alert sensitivity:**
- Detection range: Short / Medium / Long
- Alert zones: Add/edit/remove zones around body
- Directional thresholds: Adjust sensitivity per direction
- Alert types: Enable/disable specific alert classes

**Operational mode:**
- Sparse mode (default): Low power, PentaTrack only
- Dense mode: SLAM active, higher power
- Privacy mode: Cameras disabled
- Full active: All features enabled

**Connectivity configuration:**
- Wi-Fi settings: SSID, password (entered at setup)
- Cellular: Enable/disable, APN configuration
- Remote access: Enable/disable, token management
- BLE priority for direct connection

#### Remote Access

**When remote access is enabled:**

**Push notifications:**
- Critical alerts pushed immediately
- Notification contains: alert type, brief description, timestamp
- Tap notification to open app to live view

**Remote live view:**
- Access all live feeds remotely
- Quality adapts to cellular bandwidth
- Can view 360° panorama at reduced quality over LTE

**Remote recordings access:**
- Browse all recordings stored on belt node
- Download recordings remotely (bandwidth-dependent speed)
- Cannot download to device while app is closed (iOS limitation)

**Remote configuration:**
- Full configuration access
- Change operational modes
- Adjust alert thresholds

**Security for remote access:**
- Separate remote access token (distinct from local token)
- TLS required for all remote connections
- Session timeout: Configurable (default: 1 hour of inactivity)
- Rate limiting: Max 5 failed token attempts before lockout

#### Legal Export

**Export single recording:**
1. Select recording from library
2. Tap "Export" button
3. Choose export format:
   - Full quality (original file)
   - Compressed (smaller file)
4. Integrity manifest automatically included
5. Share to: Files, Email, Messaging apps, Cloud storage

**Export manifest includes:**
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

**Chain hash explanation:** Each recording has a hash that includes the hash of the previous recording, creating an immutable chain. Any tampering with a recording breaks the chain, detectable during verification.

---

## 4. Desktop App (Windows / macOS / Linux)

All mobile features plus the following additional capabilities:

### Multi-Node Recording Management

**Synchronized timeline view:**
- All nodes displayed on unified timeline
- Each node as a separate track
- Scrub through time to see what all nodes were recording simultaneously
- Click on any recording segment to play

**Annotation tools:**
- Add text notes to any point in timeline
- Mark events with custom labels
- Draw on video frames (drawing layer, non-destructive)
- Export annotations as separate file

**Search and filter:**
- Full-text search across annotations
- Filter by multiple nodes simultaneously
- Advanced filtering by event type, duration, quality

**Bulk operations:**
- Select multiple recordings
- Batch export
- Batch delete
- Batch download

### 3D World Model Viewer (Linux SoM Belt Variant)

**Interactive 3D reconstruction:**
- Full geometry of visited environments
- Objects classified and labeled
- Textured mesh from camera data

**Navigation:**
- Free-fly camera: WASD + mouse look
- First-person: Walk through as wearer would have seen
- Orbit: Rotate around specific point

**Timeline integration:**
- Scrub timeline to see world state at any past moment
- Entity trajectories displayed as animated paths
- Events annotated on timeline (alerts, detections)

**Use cases:**
- Forensic review: Replay incident in 3D context
- Security analysis: Identify blind spots, approach vectors
- Incident reconstruction: Document for legal/insurance purposes

**Export:**
- Export 3D model (OBJ, PLY formats)
- Export 360° panoramic images at specific timestamps
- Export entity trajectory data (CSV)

### Body-Frame Analytics

**Detection heatmap:**
- Top-down view of body-frame coordinate system
- Color-coded by detection frequency
- Shows where detections most commonly occur relative to wearer

**Trajectory replay:**
- Animate all detected entity trajectories over time period
- Speed control: real-time, 2x, 4x, 8x
- Filter by entity class

**Pattern analysis:**
- Detection frequency by time of day
- Detection frequency by body quadrant (front, rear, left, right)
- Correlation with wearer activity level

### Gait Reports

**Long-term trend analysis:**
- Cadence trend over weeks/months
- Asymmetry trend over time
- Impact force distribution histogram
- Stumble event frequency over time

**Comparative analysis:**
- Compare gait metrics across days
- Compare morning vs. afternoon
- Compare pre- and post-exercise

**Export:**
- PDF report with charts and statistics
- CSV export of raw gait data
- Medical-friendly format for sharing with healthcare provider

### Full Configuration Manager

**TOML editor:**
- Full `sentinel-wear.toml` editing
- Live validation (errors shown immediately)
- Schema-aware autocomplete
- Reset to defaults option

**Advanced settings:**
- BAN scheduling parameters (BLE connection intervals)
- UWB configuration
- RF coexistence settings
- Power profile customization
- Debug logging levels

**Firmware management:**
- Check for firmware updates
- Upload new firmware to belt node
- Flash firmware to individual nodes (via belt node)
- View firmware version history

**Calibration tools:**
- Re-run walk-through calibration
- Adjust node position offsets manually
- Re-run 360° pendant camera calibration
- View calibration confidence scores

### Legal Export Suite

**Full integrity export:**
- ZIP file containing: recording + integrity manifest + chain hash
- PDF report with formatted metadata
- QR code linking to verification endpoint (future feature)

**Batch export:**
- Export all recordings for a date range
- Export all recordings with specific alert type
- Generate combined manifest for batch

**Verification:**
- Verify integrity of any exported recording
- Check chain hash consistency
- Compare current hash against stored hash

### Backup and Archive

**Scheduled backups:**
- Automatic backup to network storage (NAS)
- Schedule: daily, weekly, monthly
- Retention: keep last N backups

**Archive management:**
- Archive older recordings to cold storage
- Search archived recordings
- Restore from archive

**Import:**
- Import recordings from external source
- Validate integrity on import
- Add to belt node recording database

---

## 5. Belt Node API Reference

### Authentication

All API endpoints require a bearer token set by the user:

```
Authorization: Bearer <user_configured_token>
```

**Token management:**
- Local token: Required for all local network access
- Remote token: Separate token for remote access (when enabled)
- Tokens stored hashed on belt node
- Token rotation: User can rotate tokens at any time (invalidates all active sessions)

### REST API

#### Node Management

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes` | List all nodes with health, capabilities, battery percentage |
| GET | `/api/nodes/{id}` | Single node details |
| GET | `/api/nodes/{id}/health` | Detailed health for one node |
| GET | `/api/nodes/{id}/coverage` | Coverage metrics for one node |
| POST | `/api/nodes/pair` | Start pairing mode for new node |
| DELETE | `/api/nodes/{id}` | Remove node from mesh |

#### Detection and Tracking

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/detections/recent` | Recent detections (last N, configurable) |
| GET | `/api/detections/{id}` | Single detection details |
| GET | `/api/tracks` | Currently active tracks |
| GET | `/api/tracks/{id}` | Single track details |
| GET | `/api/world-model` | Current world model state (entities, mode, SLAM available) |
| GET | `/api/world-model/replay/{timestamp_ms}` | World model state at past timestamp |

#### Alerts

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/alerts` | Alert history (paginated, filterable by type/date) |
| GET | `/api/alerts/{id}` | Single alert details |
| POST | `/api/alerts/{id}/acknowledge` | Acknowledge an alert |
| GET | `/api/extreme-velocity/events` | Fast object detection events |
| GET | `/api/extreme-velocity/events/{id}` | Single extreme velocity event details |

#### Recordings

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/recordings` | List recordings (paginated, filterable) |
| GET | `/api/recordings/{id}` | Download recording file (binary) |
| GET | `/api/recordings/{id}/meta` | Integrity manifest JSON |
| DELETE | `/api/recordings/{id}` | Delete recording and associated manifest |
| POST | `/api/recordings/search` | Advanced search with filters |

#### Export

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/export/{id}` | Export recording with integrity chain (ZIP: file + manifest) |
| POST | `/api/export/bulk` | Bulk export for date range |
| GET | `/api/export/{id}/report` | PDF report suitable for legal submission |
| GET | `/api/export/{id}/verify` | Verify recording integrity |

#### Media Streaming

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/nodes/{id}/stream` | Redirect to RTSP or WebSocket stream for this node |
| GET | `/api/nodes/{id}/panorama` | Redirect to 360° panorama stream for this node |
| GET | `/api/nodes/{id}/panorama/latest` | Latest stitched panorama frame (JPEG) |
| POST | `/api/nodes/{id}/stream/start` | Start live stream from node |
| POST | `/api/nodes/{id}/stream/stop` | Stop live stream from node |

#### Gait Analytics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/gait/history` | Gait event history (paginated) |
| GET | `/api/gait/metrics` | Current gait metrics window |
| GET | `/api/gait/stumble-events` | Stumble precursor and fall events |
| GET | `/api/gait/trends` | Long-term trend data |

#### Analytics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/analytics/body-frame` | Detection heatmap data |
| GET | `/api/analytics/body-frame/replay/{timestamp_ms}` | Entity positions at past timestamp |
| GET | `/api/analytics/gait-trends` | Long-term gait trend data |
| GET | `/api/analytics/coverage` | Node coverage analysis |

#### World Model

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/world-model` | Current world model state |
| GET | `/api/world-model/mode` | Current mode (sparse/dense) |
| POST | `/api/world-model/mode` | Switch mode |
| GET | `/api/world-model/3d` | Current 3D SLAM mesh (if available) |
| GET | `/api/world-model/replay/{timestamp_ms}` | World model at past timestamp |

#### Connectivity

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/connectivity` | Current connectivity status (WiFi, cellular, BLE) |
| GET | `/api/connectivity/wifi` | WiFi status, signal strength |
| POST | `/api/connectivity/wifi/connect` | Connect to WiFi network |
| GET | `/api/connectivity/cellular` | Cellular status, signal, carrier |
| POST | `/api/connectivity/cellular/toggle` | Enable/disable cellular module |
| GET | `/api/connectivity/cellular/data-usage` | Current billing period data usage |
| GET | `/api/connectivity/ban` | BAN network status (all nodes, sync quality) |

#### Calibration

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/calibration/neutral-pose/start` | Begin neutral-pose calibration |
| POST | `/api/calibration/neutral-pose/complete` | Complete neutral-pose calibration |
| POST | `/api/calibration/walk/start` | Begin walk-through calibration |
| POST | `/api/calibration/walk/complete` | Complete walk-through calibration |
| GET | `/api/calibration/status` | Per-node calibration state |
| POST | `/api/calibration/360-pendant/start` | Begin 360° pendant camera calibration |
| POST | `/api/calibration/360-pendant/complete` | Complete 360° pendant calibration |
| GET | `/api/calibration/360-pendant/status` | 360° pendant calibration status |

#### Configuration

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/config` | Current full configuration |
| POST | `/api/config` | Update configuration (partial or full) |
| POST | `/api/config/reset` | Reset to default configuration |
| POST | `/api/policy/mode` | Change operational mode |

#### System

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | System health summary |
| GET | `/api/health/detailed` | Detailed health per subsystem |
| GET | `/api/system/info` | Firmware version, hardware variant, uptime |
| POST | `/api/system/reboot` | Reboot belt node (requires confirmation) |
| GET | `/api/logs` | Recent system logs |
| GET | `/api/logs/download` | Download full log file |

### WebSocket Streams

| Path | Protocol | Description |
|------|----------|-------------|
| `/ws/events` | JSON | All live BAN events |
| `/ws/detections` | JSON | Live detection events only |
| `/ws/tracks` | JSON | Live track updates |
| `/ws/gait` | JSON | Live gait events |
| `/ws/alerts` | JSON | Live alert events |
| `/ws/extreme-velocity` | JSON | Live extreme velocity detections |
| `/ws/world-model` | JSON | Live world model updates (5–20 Hz) |
| `/ws/node/{id}/sensor` | Binary | Raw sensor data stream (if configured) |

**WebSocket message format:**
```json
{
  "type": "Detection",
  "timestamp_ms": 1710509422004,
  "data": {
    "track_id": "trk_0042",
    "classification": "HumanApproaching",
    "position": {"x": 3.2, "y": 1.1, "z": 0.9},
    "velocity": {"x": 0.8, "y": 1.2, "z": 0.0},
    "confidence": 0.91,
    "bearing_rad": 0.34,
    "range_m": 3.5
  }
}
```

### Media Streaming Endpoints

| Protocol | Port | Path | Description |
|----------|------|------|-------------|
| RTSP | 9090 | `rtsp://belt:9090/{node_id}` | Standard RTSP per-node stream |
| WebSocket H.264 | 9091 | `/ws/media/{node_id}` | Browser/app-compatible H.264 stream |
| WebSocket 360° | 9092 | `/ws/panorama` | Equirectangular panorama stream |
| PCM Audio | 9093 | `/ws/audio/{node_id}` | Uncompressed audio stream |
| MJPEG | 9094 | `/mjpeg/{node_id}` | MJPEG stream for simple players |

**Stream quality negotiation:**
- Client requests specific quality via query parameter: `?quality=high|medium|low`
- Server responds with best available quality at or below requested
- Quality adapts based on connection bandwidth (when adaptive mode enabled)

---

## 6. Data Flow Architecture

### Recording Capture to App Access

```
Node Sensors (pendant, bracelets, anklets, eyewear)
       │
       ▼
Node MCU Firmware (embedded)
   - Sensor capture
   - On-node processing (event detection, compression)
   - SD card storage (if configured)
       │
       ▼
BAN (BLE 5.x / UWB)
   - Metadata: BLE (low bandwidth)
   - Video burst: UWB (moderate bandwidth)
   - Live stream: BLE trigger → UWB burst
       │
       ▼
Belt Node (Linux SoM or MCU)
   - Aggregates all node data
   - SLAM processing (if enabled)
   - Recording management
   - API server
       │
       ├──► Local Storage (SD card)
       │         - Raw recordings
       │         - Compressed clips
       │         - Integrity manifests
       │         - SLAM maps
       │
       └──► Network Interface (WiFi / Cellular / BLE)
                │
                ├──► Companion App (Wi-Fi, local network)
                │         - Live streaming
                │         - Recording access
                │         - Full configuration
                │
                ├──► Companion App (Cellular, remote)
                │         - Remote streaming
                │         - Recording download
                │         - Push notifications
                │
                └──► Companion App (BLE direct, fallback)
                          - Alerts only
                          - Metadata
                          - Limited configuration
```

### 360° Panorama Data Flow

```
360° Pendant (8 cameras)
       │
       ▼
Internal Wired Bus (MIPI CSI-2)
   - Hardware FSYNC sync
   - Raw frames to vision processor
       │
       ▼
Vision Processor (on pendant)
   - Stitch to equirectangular
   - Encode (H.264/H.265)
       │
       ▼
BAN (UWB preferred for video)
   - 2K-2.5K continuous: UWB
   - 4K: Belt power + WiFi required
   - Tiered: QVGA baseline + progressive keyframes
       │
       ▼
Belt Node
   - Receives 360° stream
   - Optional re-encoding for bandwidth
   - Storage to SD card
   - Streaming to app
       │
       ▼
Companion App
   - Live 360° view
   - VR headset mode
   - Recording playback
   - Export
```

---

## 7. Integrity and Chain Hash System

### Purpose

Ensure that any exported recording can be verified as:
1. Authentic (recorded by claimed device)
2. Unmodified (not altered since recording)
3. Temporally ordered (sequence within chain is verifiable)

### Integrity Manifest Schema

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
  "chain_hash": "b7e8d1f3a2c4e5f6b7a8c9d0e1f2a3b4",
  "previous_chain_hash": "f9e8d7c6b5a4e3f2d1c0b9a8e7f6d5c4",
  "device_public_key": "-----BEGIN PUBLIC KEY-----\n..."
}
```

### Chain Hash Mechanism

```
Recording 1 → SHA-256(R1) → Chain_Hash_1 = SHA-256(R1 || null)
Recording 2 → SHA-256(R2) → Chain_Hash_2 = SHA-256(R2 || Chain_Hash_1)
Recording 3 → SHA-256(R3) → Chain_Hash_3 = SHA-256(R3 || Chain_Hash_2)
...
```

Each recording's chain hash includes the previous recording's chain hash, creating an immutable sequence.

### Verification Process

1. **User exports recording:** ZIP file contains recording + manifest
2. **User runs verification:**
   - Compute SHA-256 of recording file
   - Compare against `file_sha256` in manifest
   - Verify `chain_hash` computation using previous hash
   - Verify manifest signature using device public key (future feature)
3. **Verification result:**
   - PASS: Recording is authentic and unmodified
   - FAIL: Recording has been modified or manifest is invalid

---

## 8. Security Model

### Authentication

**Local network:**
- Bearer token required for all API endpoints
- Token set by user during initial setup
- Token stored hashed on belt node (bcrypt with work factor 12)

**Remote access:**
- Separate remote access token (user can use same or different token)
- TLS required for all remote connections
- Session timeout after 1 hour of inactivity (configurable)
- Rate limiting: 5 failed attempts before 15-minute lockout

### Encryption

**Local network:**
- TLS optional (user choice)
- Recommended for environments with shared Wi-Fi

**Remote access:**
- TLS mandatory
- Certificate pinned to belt node
- No third-party certificate authority required

### Data Protection

**In transit:**
- All API traffic over HTTPS
- Media streams over encrypted WebSocket
- Mesh traffic (BAN) signed with HMAC-SHA256

**At rest:**
- Recordings stored unencrypted by default (user's responsibility)
- Optional encryption at rest (user-configured passphrase)
- Integrity manifests always present

### Audit Logging

**All actions logged:**
- API access
- Configuration changes
- Recording access
- Export operations
- Calibration events

**Log format:**
```json
{
  "timestamp_utc": "2024-03-15T14:30:22.004Z",
  "event_type": "RECORDING_ACCESS",
  "user_token_hash": "abc123...",
  "ip_address": "192.168.1.50",
  "recording_id": "rec_0042",
  "success": true
}
```

**Log retention:** User-configured (default: 90 days)

---

## 9. Offline Operation

### Belt Node Independence

The belt node and all sensing nodes operate fully without the companion app:
- Sensing continues uninterrupted
- Recordings are stored to SD card
- Alerts are delivered via haptic/audio to wearable nodes
- All functionality preserved

### App Reconnection

When app reconnects:
- Syncs with belt node for latest state
- Downloads missed alert notifications
- Accesses all recordings captured during offline period
- No data loss

### Emergency Functionality

**If belt node is unreachable:**
- App displays last known state (cached)
- Shows "Belt Node Unreachable" indicator
- Provides option to trigger emergency contact (if previously configured with location data)

**If only BLE available:**
- App connects via BLE fallback
- Displays alerts and metadata only
- Shows "Limited Connectivity" indicator
- Full functionality restored when WiFi/cellular reconnects

---

## 10. Configuration File Example

### Companion App Configuration (`sentinel-wear.toml`)

```toml
# === COMPANION APP CONFIGURATION ===
[companion_app]
enabled = true
api_port = 8080
media_port = 9090
panorama_port = 9092

# Local network authentication
local_token = ""                    # Set during initial setup (not stored in file)
local_token_hash = ""               # Hashed token stored on belt node

# Remote access
enable_remote_access = false
remote_token = ""                   # Separate token for remote access
remote_token_hash = ""
tls_enabled = false
tls_cert_path = ""
session_timeout_minutes = 60
max_failed_attempts = 5
lockout_duration_minutes = 15

# === CONNECTIVITY ===
[connectivity]
connection_priority = ["wifi", "cellular", "ble"]
ble_fallback_enabled = true
auto_reconnect = true
connection_timeout_ms = 5000

[connectivity.wifi]
enabled = true
ssid = ""                           # Set at runtime
password = ""                       # Set at runtime (not stored)
prefer_5ghz = true

[connectivity.cellular]
enabled = false
sim_type = "nano_sim"               # "nano_sim" | "esim"
apn = ""
fallback_to_wifi = true
always_on_cellular = false
alert_via_cellular = true
stream_via_cellular = false         # Data cost consideration
emergency_contact = ""

[connectivity.ban]
primary = "ble"                     # "ble" | "uwb" | "hybrid"
ble_connection_interval_ms = 30
uwb_enabled = false
uwb_role = "timing_and_ranging"

# === DATA HANDLING ===
[data]
store_raw_video = true
store_raw_audio = false
store_raw_sensor_data = false
storage_target = "sd_card"          # "sd_card" | "internal_flash" | "network_path"
retention_days = 30                 # 0 = forever
continuous_recording = false
recording_trigger = "on_detection"  # "always" | "on_detection" | "on_alert" | "manual"
max_storage_mb = 0                  # 0 = unlimited
include_integrity_chain = true

# === STREAMING ===
[streaming]
default_quality = "adaptive"        # "high" | "medium" | "low" | "adaptive"
adaptive_bandwidth_min_mbps = 1.0
adaptive_bandwidth_max_mbps = 10.0
360_panorama_quality = "medium"

# === POWER PROFILES ===
[power]
active_mode = "standard"            # "ultra_low_latency" | "low_latency" | "standard" | "power_saving" | "sleep"
sleep_after_minutes_idle = 5
thermal_throttle_enabled = true
max_enclosure_temp_c = 40

# === ALERTS ===
[alerts]
push_notifications_enabled = true
include_location_in_notification = false
notification_minimum_urgency = "Medium"  # "Low" | "Medium" | "High" | "Critical"
cellular_alert_on_disconnect = true

# === EXTREME VELOCITY ===
[extreme_velocity]
enabled = false
mode = "production"                 # "disabled" | "research" | "production"
alert_on_detection = true
min_velocity_ms = 50
max_detection_distance_m = 10
```

---

## 11. Development Notes

### Building the App

**Mobile (iOS/Android):**
- React Native or Flutter recommended for cross-platform
- Native modules for:
  - BLE communication (BLE fallback mode)
  - Video decoding (H.264/H.265)
  - 360° panorama rendering
  - VR headset integration

**Desktop (Windows/macOS/Linux):**
- Electron or Tauri for cross-platform
- Native modules for:
  - 3D rendering (WebGL or native OpenGL)
  - Video processing
  - File system access

### Testing

**Integration tests:**
- Mock belt node API server
- Test all endpoints
- Verify streaming functionality
- Test offline behavior

**End-to-end tests:**
- Full system with physical belt node
- Test connectivity modes (WiFi, cellular, BLE)
- Verify recording capture and export
- Test alert delivery

---

## 12. Future Features (Roadmap)

### Phase 1 (Current)
- Full API implementation
- Live streaming
- Recording management
- Legal export

### Phase 2
- VR headset full support
- Advanced 3D world model navigation
- Machine learning-based pattern recognition
- Automated incident reports

### Phase 3
- Multi-wearer support (multiple SENTINEL-WEAR systems)
- Cloud backup option (user opt-in)
- Third-party integrations (home automation, security systems)
- Research data export (anonymized)

---

This completes the full `SENTINEL-WEAR/apps/README.md` specification.
