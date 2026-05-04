# Future Research Areas — SENTINEL-WEAR

## Purpose

This document maps the research domains adjacent to SENTINEL-WEAR's published wearable-sensing framework. The audience is researchers, students, and qualified licensees who want to understand the broader field of wearable awareness, distributed body-frame perception, and personal-protective-device research.

MIT-licensed, survey-paper depth. The published repository implements Tier 1 only; this document maps Tiers 2 through 8 as adjacent research domains for which the published perception layer would provide upstream input. The discussion in each tier is at the level of academic survey work — naming domains, describing architectures conceptually, citing the kinds of precedents that exist in the open literature, and identifying open research questions. None of it is build-ready specification; that boundary applies for the same reasons it applies in academic survey papers, regardless of who is writing.

---

## Tier 1 — Body-Frame Distributed Sensing (In Scope)

This is what SENTINEL-WEAR's repository develops. Body-frame sensor fusion across pendant, bracelet, belt, anklet, and (optional) eyewear nodes, with each node carrying mmWave radar, IMU, microphone array, short-range LiDAR/ToF, and environmental sensors as appropriate to form factor.

Open research questions:

**Body-frame coordinate stability.** The wearer is a non-rigid frame: torso rotates relative to head, arms move relative to torso, legs move relative to torso. Published research on multi-IMU body-pose estimation provides the framework; the SENTINEL-WEAR-specific question is how to maintain a consistent body-frame across all sensor nodes when the body itself is in motion.

**Form-factor constraints.** Jewelry-grade form factors limit power, antenna size, and acceptable heat dissipation. The research question is the minimum viable form factor for each sensor modality.

**Inter-node communication.** Body-area networking (BAN) literature provides published architectures (Bluetooth Low Energy, ultra-wideband, body-coupled communication, near-field magnetic induction). The SENTINEL-WEAR-specific question is the appropriate BAN architecture for low-latency distributed sensing.

**Battery and charging.** Published research on wearable battery life, energy harvesting (kinetic, thermal, photovoltaic), and wireless charging.

**Comfort, weight, durability, and water resistance.** Wearable human-factors research is a published domain; the SENTINEL-WEAR-specific question is the comfort envelope for jewelry-grade form factors with the relevant sensors.

**Threat-class detection accuracy.** How accurately can the body-frame perception field detect approach (human, vehicle, animal), sudden events (impact, glass break, gunshot), and falls? Published research exists for each detection class; the multi-modal fusion question is open.

**Civilian transfer.** Sports gait analysis, accessibility (haptic spatial awareness for visually impaired wearers), human-factors research, ergonomic assessment, fall-detection research, gesture-based interfaces.

This tier is in scope. The repository implements it.

---

## Tier 2 — Signal-Domain Countermeasure Research (Adjacent, Out of Scope for Implementation)

The published research on personal signal-domain countermeasures addresses several established and several emerging domains. The discussion here is at survey depth.

**Sensor dazzling at body scale.** The published counter-drone dazzling literature covers eye-safe illumination of hostile imaging sensors; the body-scale extension is an open research question. The published research addresses wavelength selection (visible-band, near-IR, retroreflective response from camera optics for cooperative direction-finding), eye-safety thresholds (IEC 60825 Class 1 limits), and angular precision (how tightly does the emission need to be aimed). The SENTINEL-WEAR-specific question is whether the body-frame perception layer can produce direction-of-arrival estimates of sufficient precision to feed a directional emitter, and whether jewelry-form-factor emitters can produce useful illumination within Class 1 limits at engagement distances. The published commercial counter-UAS dazzler literature is the closest precedent. Research extension would require optical engineering, eye-safety review, and likely regulatory engagement depending on jurisdiction.

**RF and GNSS denial in the wearer's vicinity.** The published electronic-warfare literature describes architectures for selective denial of hostile RF systems. At body scale, this is severely regulatorily constrained: most jurisdictions prohibit intentional jamming entirely (FCC explicitly in the U.S., similarly in EU, UK, and most national regulators). The published academic work is at architectural depth; operational parameters are not in the open literature. Research extension would require engagement with national spectrum regulators, which generally license such research only to specific authorities. The body-scale engineering questions — power budget, antenna size, selectivity, avoiding harm to the wearer's own devices and bystanders — have no published successful precedent at jewelry form factor.

**Acoustic deterrents.** Parametric-array acoustic literature describes architectures for narrow-beam acoustic emission, including audible-frequency parametric arrays and ultrasonic carriers. Commercial personal-safety products in this family exist. The SENTINEL-WEAR-specific question is integration with the body-frame perception layer for *targeted* (rather than omnidirectional) emission keyed to direction-of-arrival of an approaching threat. The published research is approachable; the regulatory landscape is generally permissive at hearing-safe levels; engineering questions of power, weight, and form factor at jewelry scale are open.

**Decoy emission.** The published vehicle-defense decoy literature (flares, chaff, towed decoys, RF decoys) is rich; body-scale equivalents are essentially unstudied. The published research areas would include thermal-signature decoys (small heating elements that briefly produce a thermal signature similar to the wearer's), RF decoys (small emitters that produce a more attractive RF signature than the wearer's devices), and acoustic decoys. The body-scale questions are power budget, deployment kinematics, and effectiveness against modern multi-modal sensors.

**Wearable cyber-defense.** Published research on protecting the wearer's own personal devices from compromise — not signal-domain attack on others. Includes secure-element architectures, side-channel resistance, and supply-chain integrity. Published research is mature for general electronics; the wearable-specific question is integration with the BAN.

**Information-warfare countermeasures.** Published research on protecting the wearer from social-engineering, deepfake, and impersonation attacks via the wearable's audio and information channels. Published research is at varying maturity.

This tier is documented for the research-domain map. Researchers and licensees pursuing implementation in any of these subdomains are responsible for the relevant regulatory framework. SENTINEL-WEAR's published perception layer would provide upstream input (direction-of-arrival, threat classification, timing) for any of these subdomains; implementation of the emitter or other actuator side is outside the published repository.

---

## Tier 3 — Passive Materials and Reactive Materials Research (Adjacent)

The materials-science research that surrounds personal protective wearables is a published academic and industrial domain with continuous publication.

**Bio-inspired protective fibers.** The published spider-silk literature includes mechanical characterization of orb-weaver and dragline silks (specific tensile strength comparable to or exceeding aramid fibers in some species), recombinant production approaches, and weaving-into-composites research. Practical status: spider silk has remarkable specific properties but has not yet displaced para-aramid (Kevlar) or ultra-high-molecular-weight polyethylene (UHMWPE/Dyneema) in fielded armor; research is real but several generations from popular framing.

**Para-aramid and UHMWPE composites.** Mature, fielded materials with extensive published mechanical characterization. Civilian and regulated armor categories vary by jurisdiction.

**Ceramic composite armor.** Boron carbide, silicon carbide, alumina ceramics in composite structures. Mature published material-science domain. Heavier and rigid; not directly applicable to jewelry form factor but relevant to belt-form-factor wearables.

**Shear-thickening fluids and dilatant materials.** D3O, Dilatant Foam, and similar commercial products use non-Newtonian materials whose viscosity increases under high strain rates. Used in impact-protection products. Civilian-grade and unregulated. Relevant to fall-protection and blunt-impact use cases that *are* in scope for SENTINEL-WEAR's sensing.

**Auxetic structures.** Materials with negative Poisson's ratio that contract laterally under tension. Published research on puncture resistance and impact distribution. Research-stage.

**Magnetorheological fluids.** Fluids whose viscosity changes under applied magnetic field. Published research on adaptive damping; the protective-application research is exploratory.

**Electroactive polymers.** Polymers that change shape or stiffness under applied voltage. Published research on adaptive materials; the protective-application research is exploratory.

**Phase-change materials.** Materials that absorb energy through phase transitions. Published research on thermal management and impact absorption.

**Self-healing materials.** Materials that repair small damage autonomously. Published research is mature for some chemistries.

**Metamaterials for impact protection.** Engineered structures with mechanical properties not found in natural materials. Published research at varying maturity.

The SENTINEL-WEAR connection: any reactive material would need a *trigger signal*, and the body-frame perception layer is the natural source. Whether triggering provides meaningful protective gain over passive materials at the relevant timescales is itself an open research question.

This tier is documented; researchers conducting passive-materials bench characterization may contribute relevant findings to the repository's documentation. The repository does not develop reactive or active material applications.

---

## Tier 4 — Mechanical Active Concepts (Body-Scale APS Analogues, Adjacent)

The published vehicle-scale active protection systems literature is the precedent for any body-scale equivalent. Published systems include Trophy (Israel), Iron Fist (Israel), Arena (Russia), Drozd (Russia), AMAP-ADS (Germany), and several U.S. programs at varying disclosure levels. The published architecture, at survey depth:

**Detection.** Radar (typically X-band or higher), sometimes augmented by electro-optical or acoustic sensors, detects an inbound projectile.

**Tracking.** Computes intercept geometry: where will the projectile be at engagement time, where must the interceptor be deployed.

**Decision.** A computer (or in some systems an operator) decides whether to engage based on classification, geometry, and rules-of-engagement.

**Interception.** An interceptor (small explosive payload, kinetic projectile, or directed-energy element depending on system) is deployed on a millisecond timeline.

**Effect assessment.** Sensors observe the result and update tracking.

The published vehicle-scale architecture is openly described in NATO standardization documents, academic survey papers, and defense-trade publications. Operational parameters of specific fielded systems are not generally in the open literature.

The body-scale extension is an open research domain with substantial unsolved problems:

**Standoff distance.** Vehicle-scale APS deploys interceptors at meters of standoff, providing time for blast effects to dissipate before reaching the protected platform. Body-scale has no comparable standoff. The research question is whether any interceptor mechanism can act effectively at zero or near-zero standoff without harming the protected wearer.

**Interceptor kinematics.** Vehicle-scale interceptors operate at velocities and energies suitable for projectile defeat. Body-scale equivalents would face the same kinematic requirements with vastly less power, mass, and standoff budget. The research question is whether any actuator technology — electric, pneumatic, pyrotechnic, electromagnetic, shape-memory-alloy, or otherwise — can achieve the relevant kinematics within wearable form-factor constraints.

**Threat classification at engagement timeline.** Vehicle-scale APS has milliseconds; body-scale has fewer milliseconds. The classification question (is this a real threat, is it a kinetic round, is the trajectory toward the wearer) must be answered within the engagement timeline.

**Bystander safety.** Any body-scale interceptor that deflects, captures, or destroys an inbound creates secondary projectiles or effects. The research question is whether any mechanism can bound bystander effects within acceptable limits.

**Wearer safety.** Any body-scale interceptor that operates in immediate proximity to the wearer must not harm the wearer through its own action. The research question is whether any mechanism can be made wearer-safe at the relevant energies and timescales.

**Ethical, legal, and regulatory framework.** Personal active protective devices are heavily regulated where they exist at all (mostly in military and law-enforcement contexts), and the civilian regulatory framework is largely undefined because no fielded civilian device exists.

**Conceptual mechanism families** that have appeared in the published research and patent literature, named here for completeness without engineering depth:

- *Deployable barrier concepts.* A protective surface deployed in milliseconds in front of a sensed inbound. Crash-airbag-like kinematics, but on projectile timescales rather than crash timescales. Published research is mostly in the crash-safety domain; projectile-timescale extension is largely unstudied.

- *Capture / restraint concepts.* A mechanism that captures or entangles an incoming object. Net-launcher counter-UAS systems are the closest published precedent at standoff distance; body-scale equivalents have no published precedent.

- *Kinetic interception / deflection concepts.* A mechanism that physically intercepts an incoming object's path. The energy budget, the standoff, and the wearer-safety constraints are each individually challenging; jointly, they constitute the unsolved problem. Spider-silk-style strong-fiber webbing has been informally proposed in popular discussion; the published materials research is several generations away from a wearable bullet-stopping fabric, and even a successful fabric would not address the kinematic standoff problem.

- *Reactive armor concepts.* Triggered material activation that changes the local material properties on impact prediction. Vehicle-scale reactive armor is mature; body-scale is unstudied.

- *Pyrotechnic concepts.* Small explosive interceptors, as in vehicle APS. Civilian regulatory framework essentially prohibits this at body scale in most jurisdictions, and the wearer-safety problem is severe.

- *Pneumatic and electromagnetic concepts.* Compressed-gas or electromagnetic deployment of an interceptor. Lower wearer-safety concern than pyrotechnic; energy budget at wearable scale is severely limited.

The maintainers' position is that body-scale Tier 4 is currently unsolved as a research domain — not that it is forbidden as a research subject. Researchers and licensees with the institutional, regulatory, ethical, and safety frameworks to pursue any of these subdomains are the audience for the published vehicle-APS and personal-protective-device literature; SENTINEL-WEAR's published perception layer is upstream input for any such research and is MIT-licensed for that use.

---

## Tier 5 — Energy-Domain Active Response (Adjacent)

The directed-energy analogue of Tier 4. Heavily regulated; technically dominated by the same physics constraints that make HALO-AD a simulation-only project. At body scale, additional constraints:

**Wearer proximity.** The wearer is intimately close to any wearable emitter. Eye-safety, thermal-safety, and bystander-safety problems compound beyond what any current technology can address at the energies that would be relevant for active protection.

**Power budget.** Wearable batteries cannot deliver the instantaneous power for directed-energy effects at protective thresholds.

**Form factor.** Directed-energy emitters of any meaningful capability are far larger than jewelry form factor.

The published vehicle-scale directed-energy literature (HELIOS, Iron Beam, MEHEL described in the HALO-AD future-research document) is the precedent. Body-scale extension has no near-term plausibility.

This tier is documented; the maintainers do not consider it tractable in any near-term horizon and do not develop it.

---

## Tier 6 — Bio-Integrated and Speculative Concepts (Adjacent)

Published research at varying maturity:

**Smart-textile reactive armor.** Fabrics with embedded sensing and reactive elements. Published research is exploratory.

**Bio-integrated sensing.** Sensors integrated with the wearer's own biology (skin-mounted, tattoo-form-factor electronics, ingestible sensors, implantable sensors). Published research is mature for medical applications and exploratory for other applications.

**Programmable matter.** Materials that change shape or properties under control. Published research is at academic-prototype level.

**Bio-inspired distributed sensing.** Sensor architectures modeled on biological sensory systems (skin, hair-cell vibration sensing, vibrissae). Published research is exploratory.

**Closed-loop biometric integration.** Wearables that close a loop with the wearer's nervous system — for example, haptic feedback that integrates with proprioception. Published research is exploratory.

This tier is documented for completeness. The maintainers do not develop it.

---

## Tier 7 — Civilian Transfer Domains (In Scope at the Algorithmic Level)

The body-frame perception layer of SENTINEL-WEAR transfers directly to several civilian research domains:

**Sports and athletic performance.** Gait analysis, motion-pattern characterization, fatigue detection, technique analysis. Published research is mature; SENTINEL-WEAR's body-frame fusion provides a research substrate.

**Accessibility.** Haptic spatial awareness for visually impaired wearers. Published research is mature; the body-frame perception layer with directional haptic alerts is a natural research substrate. This is one of the most promising civilian applications and the maintainers actively encourage research extension in this direction.

**Workplace safety.** Industrial environments where wearers face hazards (construction, heavy industry, chemical handling). Body-frame approach detection, fall detection, and environmental hazard alerting. Published research is mature for individual hazard classes; multi-hazard fusion is an open research question.

**Elderly care and aging-in-place.** Fall detection, gait-deterioration tracking, daily-activity baselines, anomaly detection. Published research is mature; the wearable-form-factor extension is an open research question. The medical-device disclaimer applies.

**Pediatric safety.** Wearable awareness for children — without the privacy issues of camera-based solutions. Published research is preliminary; the form-factor and human-factors questions are particularly important for pediatric applications.

**Medical research instrumentation.** Body-frame perception as a research instrument in clinical studies (gait, balance, motion-pattern analysis). Published research is mature; the SENTINEL-WEAR contribution is the open framework.

**Search-and-rescue.** Wearable awareness for first responders. Published research addresses the specific environmental conditions (smoke, debris, low-light); the body-frame perception layer is a research substrate.

This tier is in scope. Civilian-transfer examples will be added to the repository's `examples/` directory.

---

## Tier 8 — Human-Computer Interaction and Interface Research

The output side of SENTINEL-WEAR — how the perception layer communicates with the wearer — is itself a research domain. Published HCI research addresses:

**Haptic-modality information density.** How much information can be conveyed through directional haptic alerts? Published research addresses pattern-based encoding, location-based encoding, and multi-actuator coordination.

**Audio-modality information density.** Bone-conduction and earpiece-based audio. Published research addresses spatial audio, audio-icon design, and attention.

**Visual-modality information density.** Companion-app and smartwatch displays. Published research addresses glanceable interfaces, peripheral-vision interfaces, and notification design.

**Multi-modal coordination.** How haptic, audio, and visual modalities should be combined and prioritized. Published research is at varying maturity.

**Attention and cognitive load.** A wearable that demands too much attention is worse than no wearable at all. Published research addresses attention budgets and notification fatigue.

**Trust and reliance.** How wearers should and do calibrate their trust in wearable awareness systems. Published research is at varying maturity.

**Privacy and social acceptability.** Wearables in public spaces. Published research addresses social acceptability of various form factors and the bystander-privacy implications of various sensors.

This tier is in scope.

---

## Summary

SENTINEL-WEAR's repository implements Tier 1, develops Tier 7 (civilian transfer) and Tier 8 (HCI), and conducts passive-material bench research at Tier 3. Tiers 2, 4, 5, and 6 are documented as adjacent research domains; the published perception layer would provide upstream input to research in those tiers conducted by qualified parties under appropriate frameworks.

All published material is MIT-licensed.
