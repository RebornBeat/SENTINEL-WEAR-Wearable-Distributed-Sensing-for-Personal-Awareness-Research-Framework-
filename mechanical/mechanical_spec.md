# Mechanical Specification — SENTINEL-WEAR Enclosures

**Version:** 0.1 | **Status:** Reference Design

---

## 1. Overview

SENTINEL-WEAR enclosures must satisfy jewelry-grade aesthetic requirements while housing sensing PCBs, batteries, and optional camera modules. The design philosophy: components follow the jewelry form, not the other way around.

---

## 2. Node Enclosures

### 2.1 Pendant — Standard (Variant A)

**Target form:** Medallion, flat pendant, or lozenge. ~50 mm × 50 mm × 15 mm maximum.

**Material:** Titanium Grade 5 outer shell with polycarbonate sensor window on front face. Silicone gasket for IP54+.

**Sensor apertures:**
- mmWave: RF-transparent front face (ABS or polycarbonate — no metal in aperture zone)
- Microphone: PTFE membrane acoustic port (4 positions, matching array geometry)
- Camera (if installed): optical-quality clear polycarbonate window
- Environmental: micro-ventilation slots on bottom edge

**Attachment:** Standard necklace bail (top center) for chain/cord attachment.

### 2.2 Pendant — 360° Curved (Variant B)

**Target form:** Arc-shaped pendant, 180–360° arc following necklace curve. Approximately 120–160 mm arc length × 15 mm wide × 10–15 mm thick.

**Material:** Rigid-flex PCB arc enclosed in a curved titanium or surgical steel channel. Transparent polycarbonate windows at each camera position. Inner (body-contact) surface: medical-grade silicone.

**Camera apertures:** One optical-quality window per camera position, aligned with camera lens. Window material: AR-coated polycarbonate for clarity.

**Attachment:** Chain passes through or loops around the arc ends. Two attachment points for stability.

**Balance:** Battery distribution along the arc or in a separate module on the chain itself.

**CAD:** `mechanical/cad/pendant_360_arc.step`, `mechanical/cad/pendant_360_full_assembly.step`

### 2.3 Pendant — Medallion (Variant C)

**Target form:** Larger circular or hexagonal medallion. 60 mm diameter × 20–25 mm thick.

**Material:** Same as Variant A, but with larger enclosure volume for higher-capacity battery and compute.

### 2.4 Bracelet Nodes (Left and Right)

**Target form:** Wristwatch-style band with dorsal module. PCB and battery in the dorsal case; band wraps around wrist.

**Case dimensions:** 40 mm × 35 mm × 8–12 mm (dorsal profile). Band width: 22 mm standard.

**Material:** Grade 5 titanium or surgical steel case. Medical-grade silicone band. IP67 sealing.

**Sensor apertures:**
- mmWave: outward-facing (dorsal → away from wrist). RF-transparent material in aperture zone.
- Haptic: actuator internal, vibration transmitted through case bottom (inner wrist contact).
- Camera (Variant C): small optical window on outward dorsal face.

**Band attachment:** Standard 22 mm quick-release spring bars. Compatible with standard watch bands.

**Charging:** Magnetic POGO pin contacts on case bottom (inner wrist surface).

### 2.5 Belt Node

**Target form:** Belt buckle module or rear belt clip.

**Belt Buckle Option:** 80 mm × 60 mm × 20 mm brushed aluminum or titanium. Integrated buckle mechanism replaces standard buckle. PCB and battery behind the buckle plate.

**Belt Clip Option:** 100 mm × 70 mm × 25 mm rear clip pack. Clips to existing belt. More internal volume for larger battery.

**Material:** Anodized aluminum (heat dissipation). Ventilation slots on non-contact surfaces.

**Charging:** USB-C port accessible on side or bottom edge.

**CAD:** `mechanical/cad/belt_node_buckle.step`, `mechanical/cad/belt_node_clip.step`

### 2.6 Anklet Nodes (Left and Right)

**Target form:** Slim sports band or jewelry anklet band with small module.

**Module dimensions:** 40 mm × 30 mm × 12 mm. Secured to upper ankle.

**Material:** Surgical steel module with silicone strap. IP67 sealing.

**Sensor apertures:**
- ToF/LiDAR: forward-facing window.
- IMU: internal (no aperture needed).
- Haptic: internal.
- Camera (Variant B): small optical window on forward-facing surface.

### 2.7 Eyewear Node

**Clip-on (Variants A, B):** 30 mm × 20 mm × 8 mm. Clips to frame bridge or front. Weight: target < 8 g.

**Frame-integrated (Variant C):** Segments embedded in temple arms. Each arm segment: 60 mm × 4 mm × 3 mm. Requires custom or semi-custom frames. Material: lightweight PETG or ABS for prototype; aluminum for production.

---

## 3. Hypoallergenic Compliance

All skin-contact surfaces comply with EN 1811:2011 nickel release requirements. See `hardware/hardware_config.md` Section 10 for compliant materials list.

---

## 4. Water Resistance Standards

| Node | Target Rating | Test Method |
|---|---|---|
| Pendant | IP67 | 1 m submersion, 30 min |
| Bracelets | IP67 | 1 m submersion, 30 min |
| Belt Node | IP54 | Splash test |
| Anklets | IP67 | 1 m submersion, 30 min |
| Eyewear (clip-on) | IPX4 | Sweat / splash |
| Eyewear (frame-integrated) | IPX4 | Sweat / splash |

---

## 5. CAD Files

```
mechanical/cad/
├── pendant_std_enclosure.step
├── pendant_360_arc.step
├── pendant_360_full_assembly.step
├── pendant_medallion_enclosure.step
├── bracelet_case_22mm.step
├── belt_node_buckle.step
├── belt_node_clip.step
├── anklet_module_enclosure.step
├── eyewear_clipon.step
├── eyewear_frame_arm_segment.step
└── eyewear_headband.step
```

STL exports for 3D printing in `mechanical/stl/`.
