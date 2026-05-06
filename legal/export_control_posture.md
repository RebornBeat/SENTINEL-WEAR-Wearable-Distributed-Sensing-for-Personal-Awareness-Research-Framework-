# Export Control Posture — SENTINEL-WEAR

**Version:** 1.0

---

## 1. Summary

SENTINEL-WEAR is a general-purpose wearable sensing research platform. The maintainers do not believe any component of SENTINEL-WEAR as designed and published constitutes a controlled item under EAR, ITAR, or EU Dual-Use Regulation.

**This is the maintainers' good-faith opinion, not legal advice.** Deployers conducting cross-border shipments should perform their own export control review.

---

## 2. Component Analysis

### 2.1 mmWave Radar (60 GHz)

Consumer/industrial 60 GHz radar modules. Unlicensed frequency band. Widely exported for automotive and consumer applications. Likely EAR99.

### 2.2 BLE / UWB Radio

Standard commercial wireless technology. UWB modules (DW3000) are commercially available for consumer and industrial applications globally. Likely EAR99.

### 2.3 Cameras and Biometric Sensors

Standard commercial image sensors (OmniVision, Sony, HiMax). No export control parameters expected. Verify manufacturer ECCN classification.

### 2.4 Software (Open-Source)

Publicly available open-source software is generally exempt from EAR technology controls (§ 734.3(b)(3)). SENTINEL-WEAR, OMNI-SENSE, and PentaTrack are all publicly available open-source projects.

---

## 3. Prohibited Use

SENTINEL-WEAR should not be deployed for any application that triggers export control obligations, including:
- Military sensing applications.
- Intelligence gathering applications.
- Any defense-sector application in a jurisdiction requiring export licenses.

---

## 4. Cross-Border Hardware Shipment

Before shipping SENTINEL-WEAR prototype hardware across national borders:
- Verify the destination is not subject to comprehensive trade embargo.
- Confirm with each module manufacturer that their specific component is authorized for export to the destination.
- Consult a licensed export control attorney if uncertain.
