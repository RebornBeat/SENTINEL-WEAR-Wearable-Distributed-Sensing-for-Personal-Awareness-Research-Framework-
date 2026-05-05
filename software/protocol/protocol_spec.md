# Body-Area Mesh Protocol Specification — SENTINEL-WEAR

## Scope

This document specifies the protocol used among SENTINEL-WEAR's nodes (pendant, bracelet, belt, anklet, eyewear) and between nodes and the wearer's companion device (phone, smartwatch, desktop). The protocol is designed for low-latency body-area distributed sensing.

The protocol carries sensor metadata, fusion outputs, alerts, and control messages. It does not carry imagery (none is captured), does not carry commands to active emitters (none are present), and does not carry commands to actuators that act on external objects (none are present).

## Physical Layer

The reference implementation uses Bluetooth Low Energy 5.x for inter-node communication, with experimental support for Ultra-Wideband (UWB) and body-coupled communication (BCC) for research purposes. Companion-device communication uses BLE for phone/smartwatch and USB or BLE for desktop.

## Data Link Layer

Each message is signed with a per-device key established during pairing. Replay protection uses sequence numbers and timestamps. Authentication uses standard Bluetooth pairing protocols supplemented by application-layer authentication for high-stakes messages (configuration changes, identity-layer enable/disable).

## Message Types

**Sensor metadata.** Per-node detection events, track updates, environmental readings.

**Fusion output.** Belt-controller-produced body-frame perception updates, threat-classification, anomaly flags.

**Alert.** Haptic-actuator commands, audio-alert commands, companion-device notification commands.

**Configuration.** Node-configuration changes, calibration data, BAN-key management.

**Audit.** Logging events for the user-inspectable audit log.

**Health.** Node battery state, sensor-fault reports, communication-quality metrics.

## What the Protocol Does Not Carry

- Imagery (no camera modalities).
- Raw audio waveforms (audio is processed locally; only event classifications and direction-of-arrival metadata are transmitted).
- Commands to active emitters above standard haptic/audio/indicator functions.
- Commands to actuators that act on external objects.
- Operational-engagement commands of any kind.

The protocol's message vocabulary is bounded such that extension to operational active-response use is not possible without redesigning the protocol entirely.

## Versioning

The protocol is versioned with the repository. Changes require a maintainer review and a CHANGELOG entry tagged `[protocol]`.
