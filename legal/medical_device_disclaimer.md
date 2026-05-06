# Medical Device Disclaimer — SENTINEL-WEAR

**Version:** 1.0

---

## Not a Medical Device

SENTINEL-WEAR is a research and education platform for distributed wearable sensing. It is not a medical device, not FDA-cleared (US), not CE-marked as a medical device (EU), and not certified under any medical device regulatory framework in any jurisdiction.

---

## Specific Non-Medical Uses

The following SENTINEL-WEAR features may appear similar to medical monitoring systems but are explicitly not medical:

### Gait Analysis (Anklet IMU)
SENTINEL-WEAR's gait pipeline computes step frequency, stride length, heel-strike impact, and stumble precursors. This information is produced for research and general awareness purposes. It is not equivalent to clinical gait analysis and should not be used for:
- Diagnosis of neurological, musculoskeletal, or balance disorders.
- Monitoring of rehabilitation progress in a clinical context.
- Any clinical decision-making.

### Fall Prediction (Stumble Precursor Algorithm)
The stumble precursor algorithm identifies patterns in IMU data that may precede stumbles. This algorithm has not been clinically validated and has an unknown false positive and false negative rate in the general population. Do not rely on this algorithm as a primary fall prevention system for individuals with fall risk.

### Environmental Monitoring (Atmospheric Sensors)
Temperature, humidity, pressure, and VOC sensors are research-quality components with general consumer accuracy specifications. Do not use for:
- Medical air quality assessment.
- Occupational health monitoring in regulated environments.
- Any application requiring certified sensor calibration.

---

## Appropriate Use

SENTINEL-WEAR is appropriate for:
- Research into wearable sensing architectures and body-frame fusion.
- Personal awareness and situational information (not medical diagnosis).
- Education and demonstration of distributed sensor systems.
- Gait data collection for research datasets (with appropriate IRB approval).

---

## Responsible Disclosure

If you observe that SENTINEL-WEAR sensor outputs contain systematic errors that could mislead users into health-affecting decisions, please report this to the project maintainers via the project's issue tracker.
