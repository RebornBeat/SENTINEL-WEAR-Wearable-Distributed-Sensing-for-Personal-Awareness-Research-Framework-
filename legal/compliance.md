# Legal Compliance Reference — SENTINEL-WEAR

**Version:** 1.0 | **Status:** Research Platform Reference

---

## 1. Scope

This document covers compliance considerations for the SENTINEL-WEAR distributed wearable sensing platform. SENTINEL-WEAR is a research platform and is not a certified commercial product. Deployers are responsible for all regulatory compliance in their jurisdiction.

---

## 2. Radio Frequency Compliance

### 2.1 Body-Area Network Radio (BLE 5.x)

BLE modules must carry applicable regional RF certification:
- **FCC (US):** Part 15, Subpart C certification. Wearable BLE devices may require specific SAR (Specific Absorption Rate) testing for body-worn operation. Verify with the selected BLE module's FCC ID documentation.
- **CE (EU):** RED (Radio Equipment Directive 2014/53/EU). SAR compliance per EN 50663 for wearable devices in close contact with the body.
- **ISED (Canada):** IC certification.

**SAR compliance is particularly important for wearable devices** because they are worn continuously in close proximity to the body. Use only certified BLE modules from suppliers who have conducted body-worn SAR measurements.

### 2.2 UWB (If Configured)

UWB operation (Qorvo DW3000 class) requires specific frequency band authorization:
- **FCC:** Part 15, Subpart F (UWB Devices). Maximum power spectral density limits apply.
- **EU:** ETSI EN 302 065 harmonized standard under RED.
- **Other regions:** UWB is not globally harmonized. Check local regulations before enabling UWB in any deployment.

### 2.3 mmWave Radar (60 GHz)

Same compliance considerations as AEGIS-MESH (see AEGIS-MESH `legal/compliance.md`, Section 2.3). Wearable mmWave radar in a pendant or bracelet position is at closer body proximity than infrastructure nodes — verify SAR compliance if applicable regulations cover this configuration.

### 2.4 Cellular Module (If Configured)

LTE/5G cellular modules for the belt node require certification from the applicable carrier and regulatory body:
- FCC ID for the specific module model.
- Carrier approval (some carriers require device certification for network access).
- SAR testing for body-worn cellular devices — belt position is at waist, typically moderate distance from sensitive anatomy.

---

## 3. Hypoallergenic and Materials Compliance

All skin-contact materials must comply with **EN 1811:2011** (nickel release requirements for articles intended to come into direct and prolonged contact with the skin).

**Required testing:** Nickel release test per EN 1811 for any metal component in skin contact. Acceptable limit: 0.2 μg/cm²/week for prolonged skin contact.

**Compliant materials list:** See `hardware/hardware_config.md` Section 10.

**Testing laboratory:** Independent EN 1811 testing is required before any commercial deployment of hardware intended for skin contact. Research prototypes should use compliant materials from the specification list to avoid skin sensitization.

---

## 4. Recording and Privacy Laws

SENTINEL-WEAR can be configured to capture audio, video, and biometric data from cameras mounted on the pendant, bracelets, anklets, and eyewear. Deployers are fully responsible for compliance.

### 4.1 Consent Requirements

Recording of individuals requires consent in most jurisdictions. Body-worn cameras recording public spaces are subject to varying legal frameworks:
- **One-party consent states/countries:** The wearer's consent is sufficient for recordings where the wearer is a participant.
- **Two-party consent states/countries:** All parties being recorded must consent.
- **Public space recording:** In many jurisdictions, recording in public is generally lawful, but covert recording of conversations may not be.
- **Private spaces:** Recording in private spaces owned by others requires explicit consent.

**The system does not enforce consent requirements.** Deployers are responsible for obtaining required consents.

### 4.2 Biometric Data

When identification camera features are used:
- Many jurisdictions classify biometric data (facial recognition, voice patterns) as special category or sensitive personal information requiring explicit consent and a lawful processing basis.
- BIPA (Illinois, US), GDPR (EU), CCPA (California, US) and similar laws may apply.
- The SENTINEL-WEAR system does not transmit biometric data externally unless the user configures streaming or remote access. All processing is local by default.

---

## 5. Not a Medical Device

SENTINEL-WEAR is not certified as a medical device:
- **Gait analysis:** Produced by the anklet IMUs. Not certified for clinical diagnostic purposes.
- **Stumble prediction:** Research algorithm. Not certified as a fall prevention system.
- **Environmental sensing:** Research quality. Not certified for health-critical air quality monitoring.

Do not make health-related decisions based solely on SENTINEL-WEAR sensor data. Consult qualified healthcare professionals for health concerns.

---

## 6. Not Certified Personal Protective Equipment

SENTINEL-WEAR is not PPE:
- Not certified under any PPE standard (ANSI, EN 361, etc.).
- Not rated for protection against physical threats.
- Not a substitute for professional security services, law enforcement response, or situational awareness training.

---

## 7. Battery Safety

All lithium cells must be certified to UL 1642 or IEC 62133. Hardware protection circuits are mandatory. The firmware provides secondary voltage cutoffs. See `hardware/hardware_config.md` Section 9.

---

## 8. Water Resistance

IP ratings are tested according to IEC 60529. IP67-rated enclosures are tested to 1 m depth, 30 minutes. This rating does not cover: high-velocity water jets, hot water, chlorinated pool water (without additional testing), or saltwater immersion. Test data for the specific enclosure must be obtained before making IP claims.

---

## 9. Export Control

Same posture as AEGIS-MESH. SENTINEL-WEAR uses commercially available sensing components in generally unlicensed frequency bands. Open-source software with publicly available exemption. Deployers should verify before cross-border shipment. See `legal/export_control_posture.md`.

---

*This document is informational only and does not constitute legal advice.*
