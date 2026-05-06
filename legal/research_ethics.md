# Research Ethics — SENTINEL-WEAR

**Version:** 1.0

---

## 1. Project Ethics Posture

SENTINEL-WEAR is a research platform for wearable distributed sensing. The project studies the feasibility of jewelry-form-factor sensing for personal awareness. The maintainers are committed to responsible development and transparent documentation of all capabilities and limitations.

---

## 2. Scope Boundary

SENTINEL-WEAR is strictly bounded to sensing and information:
- No actuation, no effectors, no physical response mechanisms.
- No mechanisms designed to intercept, deflect, or affect external objects.
- No "active protection" of any kind.

The sensing-only scope is maintained because: physical interception at personal scale is dominated by passive armor (mature, regulated, well-served); active protection devices are heavily regulated; informal active protection research poses genuine safety risks. See `docs/theory/why_no_actuation.md`.

---

## 3. Privacy and Consent

### 3.1 Wearer's Data

The wearer of a SENTINEL-WEAR system is the primary data subject. All data collected is owned by the wearer. The system does not transmit data to any external party without explicit user configuration.

### 3.2 Third-Party Data

When SENTINEL-WEAR cameras or acoustic sensors capture data about persons other than the wearer:
- Deployers are responsible for compliance with applicable recording consent laws.
- The system does not impose consent requirements — this is a research platform.
- Researchers using SENTINEL-WEAR to collect data involving human subjects must obtain appropriate ethics board approval (IRB in the US, Ethics Committee in the EU, etc.).

### 3.3 Research Data Publication

When publishing research data collected with SENTINEL-WEAR:
- Anonymize all data that could identify specific individuals.
- Do not publish recordings of individuals who have not provided informed consent.
- Comply with applicable data protection regulations for the research context.

---

## 4. Dual-Use Considerations

SENTINEL-WEAR sensing capabilities (presence detection, tracking, acoustic classification, visual identification) could be misused for covert surveillance. The maintainers condemn such use. The system is designed for owner-worn sensing of the wearer's own environment, not covert surveillance of others.

---

## 5. Extreme Velocity Sensing Research Track

SENTINEL-WEAR includes a research track on sensing physics for fast-moving objects at body-frame timescales (`docs/theory/extreme_velocity_sensing.md`). This research characterizes what sensors can detect — it does not design interception systems. The research track is explicitly upstream-of-actuation work. Any downstream actuation questions are addressed to qualified research parties operating under appropriate institutional oversight.

---

## 6. Disclosure Policy

Security vulnerabilities (particularly those enabling unauthorized access to live feeds or recordings) should be reported to the project maintainers. Commit to: acknowledgment within 7 days, fix within 30 days for critical issues, public advisory on fix release.
