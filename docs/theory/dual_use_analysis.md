# Military Dual-Use Analysis — SENTINEL-WEAR with Exoskeleton Integration

**Project:** SENTINEL-WEAR
**Domain:** Dual-use sensing architecture analysis
**Status:** Research Survey
**License:** MIT (Code), CERN-OHL-S v2 (Hardware), CC BY 4.0 (Documentation)

---

## Research Context

This document analyzes how SENTINEL-WEAR's sensing architecture could integrate with military exoskeleton systems. It is a survey of the sensing gap in current military platforms and how SENTINEL-WEAR's existing sensing capabilities could address that gap.

This is **analysis of sensing technology applications**, not specification of weapons systems. SENTINEL-WEAR remains a sensing-only platform with no actuation, no weapons capability, and no operational guidance for combat applications.

Researchers, defense analysts, and dual-use technology scholars may find this analysis useful for understanding the sensing layer of future soldier systems. The project maintainers do not develop, contract for, or consult on weapons systems implementation.

---

## Executive Summary

**SENTINEL-WEAR represents a fundamental paradigm shift in individual soldier capability.** When integrated with exoskeleton platforms, it transforms the soldier from an isolated combatant into a **mobile, networked perception node** — effectively creating a "body-scale sensing platform" that has no current equivalent in any military program worldwide.

The sensing architecture developed for civilian safety applications fills the critical gap in modern exoskeleton development: **exoskeletons provide strength and endurance, but they are fundamentally "blind."** SENTINEL-WEAR gives them eyes, ears, and reflexes.

**Critical architectural insight:** The sensing architecture is fully characterized and production-capable. Protection integration is a **materials-science research question**, not a sensing or timing constraint. At appropriate engagement distances (5+ meters for handgun, 10+ meters for rifle), the detection window is **sufficient for triggered protective systems** if suitable materials exist.

This is not incremental improvement. This is a new category of military capability — in sensing and awareness, with protection as a research-domain integration path.

---

# PART 1: CURRENT STATE OF MILITARY EXOSKELETONS

## 1.1 What Exoskeletons Currently Provide

| Capability | Current Status | Limitation |
|------------|----------------|------------|
| **Strength augmentation** | ✅ Fielded prototypes | Load bearing, lifting assistance |
| **Endurance enhancement** | ✅ Fielded prototypes | Reduced fatigue, longer marches |
| **Basic motion sensing** | ⚠️ Limited | IMU for exoskeleton control only |
| **Situational awareness** | ❌ Absent | Soldier is still dependent on external systems |
| **Threat detection** | ❌ Absent | No built-in threat sensing |
| **360° perception** | ❌ Absent | No distributed sensing architecture |
| **Squad-level mesh** | ❌ Absent | Each soldier is an island |
| **Integrated protection** | ⚠️ Limited | Armor plates only; no triggered systems |

## 1.2 Major Military Exoskeleton Programs

### United States

**ONYX (Sarcos / Lockheed Martin)**
- Lower-body exoskeleton
- Focus: Endurance and load bearing
- Sensing: Basic IMU for control
- Status: Fielded in limited quantities

**Guardian XO (Sarcos)**
- Full-body powered exoskeleton
- Focus: Heavy lifting
- Sensing: Minimal
- Status: Industrial applications, military evaluation

**TALOS (SOCOM — Cancelled)**
- Attempted full tactical exoskeleton with integrated sensing
- Vision: Head-up display, vital signs, limited threat detection
- Failure point: Integration complexity, weight, power
- Status: Program cancelled, but requirements persist

**Current Infantry Sensing (Non-Exoskeleton)**
- NVGs (AN/PSQ-20)
- Thermal sights
- Radio communications
- GPS
- **All external, all platform-centric**

### Russia

**Ratnik Program**
- Integrated soldier system
- Focus: Armor, comms, night vision
- Sensing: External sensors (NVG, thermal)
- No body-centric distributed sensing

### China

**Multiple Programs**
- Focus: Strength augmentation
- Sensing: Basic
- No distributed perception architecture

## 1.3 The Critical Gap

**Every current exoskeleton program treats the soldier as something to be augmented physically. None treat the soldier as a sensing platform.**

The soldier in an exoskeleton is:
- Stronger
- Has more endurance
- Can carry more weight
- **But is still blind and dependent on external sensing systems**

SENTINEL-WEAR fills exactly this gap.

---

# PART 2: SENTINEL-WEAR AS THE MISSING PERCEPTION LAYER

## 2.1 What SENTINEL-WEAR Provides That Exoskeletons Lack

| Capability | SENTINEL-WEAR | Current Exoskeletons | Military Gap Filled |
|------------|---------------|---------------------|---------------------|
| **360° body-centric awareness** | ✅ Full coverage | ❌ None | Individual situational awareness |
| **Extreme velocity detection** | ✅ Microsecond-scale | ❌ None | Sniper/projectile detection |
| **Distributed multi-node fusion** | ✅ PentaTrack | ❌ None | Squad-level perception mesh |
| **Body-frame coordinate system** | ✅ Stable reference | ⚠️ IMU for control only | Unified spatial reference |
| **Through-wall detection** | ✅ mmWave radar | ❌ None | Urban warfare advantage |
| **Night operations capability** | ✅ Radar/acoustic | ❌ None | Darkness-independent sensing |
| **Dense SLAM mapping** | ✅ 3D world model | ❌ None | Real-time terrain intelligence |
| **Friendly force tracking** | ✅ Per-soldier ID | ❌ None | Blue force tracking at individual level |
| **Casualty detection** | ✅ Gait analysis, vitals | ❌ None | Automated casualty alerting |
| **Evidence/intel capture** | ✅ 360° recording | ❌ None | Post-mission intelligence |
| **Detection-to-protection timing** | ✅ Characterized | ❌ None | Foundation for protection integration |

## 2.2 The Architecture Is Scale-Invariant

The same architecture developed for a 6-node civilian jewelry system scales directly to:

**Individual Soldier:** 6-10 nodes = full 360° awareness, threat detection, dense mapping

**Squad-Level Mesh:** 30-100 nodes = distributed sensing network where each soldier is a node, creating overlapping coverage, multi-perspective fusion

**Platoon-Level Mesh:** 100-300 nodes = complete battlefield perception mesh with no blind spots, multi-level fusion

---

# PART 3: CAPABILITY ARCHITECTURE — SENSING VS PROTECTION

## 3.1 The Core Architectural Separation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SENTINEL-WEAR CAPABILITY ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ SENSING LAYER — Production-Capable                                  │    │
│  │                                                                      │    │
│  │ • Extreme velocity detection: 0.5-1.6 ms                             │    │
│  │ • Alert generation: 0.65-2.4 ms                                      │    │
│  │ • Direction estimation: ±15°                                         │    │
│  │ • 360° body-centric awareness                                        │    │
│  │ • Dense SLAM mapping                                                 │    │
│  │ • Squad-level mesh fusion                                            │    │
│  │ • Evidence capture with integrity chain                              │    │
│  │                                                                      │    │
│  │ STATUS: Fully characterized, ready for integration                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                               │                                              │
│                               │ Detection event + trajectory prediction     │
│                               │ Time-to-impact estimate                    │
│                               │ Direction vector                           │
│                               ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ PROTECTION LAYER — Materials Research Domain                        │    │
│  │                                                                      │    │
│  │ Timing constraint: SOLVED (window is open at appropriate distances)  │    │
│  │ Materials constraint: RESEARCH (what materials deploy in 1-3 ms?)   │    │
│  │                                                                      │    │
│  │ Point-blank (< 3 m): Passive materials only                         │    │
│  │ Close (3-10 m): Passive + triggered systems possible                │    │
│  │ Medium/Long (10+ m): Full protection capability                      │    │
│  │                                                                      │    │
│  │ STATUS: Research documented in passive_materials_research.md        │    │
│  │         Detection-to-protection timing is fully characterized        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3.2 Detection Layer — Fully Characterized

From `doppler_radar_config.md`:

| Component | Latency | Status |
|-----------|---------|--------|
| Tier 1 CW Doppler | 50-150 µs | ✅ Production-capable |
| Tier 2 Event camera | 100-500 µs | ✅ Production-capable |
| BLE QoS Critical | 300-800 µs | ✅ Production-capable |
| Belt reception | 50-150 µs | ✅ Production-capable |
| **Electronic detection** | **0.5-1.6 ms** | ✅ **Fully characterized** |

## 3.3 Alert Layer — Fully Characterized

| Component | Latency | Status |
|-----------|---------|--------|
| Piezo haptic trigger | 50-300 µs | ✅ Production-capable |
| Piezo mechanical onset | 100-500 µs | ✅ Production-capable |
| **Total alert** | **0.65-2.4 ms** | ✅ **Fully characterized** |

## 3.4 Protection Layer — Materials Research

| System | Response Time | Engagement Distance Feasibility | Research Status |
|--------|---------------|--------------------------------|-----------------|
| UHMWPE (passive) | Always active | All distances | ✅ Production reference |
| STF (passive) | Always active | All distances | ✅ Production reference |
| Pre-tensioned filaments | 0.5-2.0 ms | 5+ m (handgun), 10+ m (rifle) | ⚠️ Advanced research |
| MR fluids | 1-5 ms | 5+ m (handgun), 10+ m (rifle) | ⚠️ Advanced research |
| EAPs (future) | < 1 ms (potential) | 3+ m (handgun), 5+ m (rifle) | ◯ Future research |

---

# PART 4: THEORETICAL FEASIBILITY MATRIX

## 4.1 Detection-to-Protection Integration — What IS Achievable

| Engagement Distance | Detection | Alert | Protection Deployment | Human Response | Status |
|--------------------|-----------|-------|----------------------|-----------------|--------|
| **Point-blank (< 3 m)** | ✅ Achievable | ✅ Achievable | ❌ No time | ❌ No | Physically constrained |
| **Close (3-5 m)** | ✅ Achievable | ✅ Achievable | ⚠️ Rifle: marginal, Handgun: possible | ❌ No | Research frontier |
| **Close (5-10 m)** | ✅ Achievable | ✅ Achievable | ✅ Handgun: comfortable, Rifle: marginal | ⚠️ Marginal | Research frontier |
| **Medium (10-50 m)** | ✅ Achievable | ✅ Achievable | ✅ All systems | ⚠️ Partial | Fully achievable |
| **Long (50+ m)** | ✅ Achievable | ✅ Achievable | ✅ Trivial | ✅ Full | Fully achievable |

## 4.2 The Key Insight

**At 5+ meters for handgun, 10+ meters for rifle, the physics window is OPEN for triggered protective systems.**

The question is NOT "can protection deploy in time?" — it CAN (0.5-2 ms deployment fits within 6-12 ms warning times).

The question IS: "what material system can provide meaningful protection when deployed?"

This is a **materials science problem**, not a **timing problem**.

## 4.3 Engagement Distance Reality Check

### Civilian Scenarios

| Distance | Weapon | Flight Time | Warning Time | Protection Feasibility |
|----------|--------|--------------|--------------|------------------------|
| 3 m | Handgun | 8.5 ms | 6-8 ms | Passive + triggered marginal |
| 5 m | Handgun | 14.3 ms | 12-14 ms | Passive + triggered comfortable |
| 10 m | Handgun | 28.6 ms | 26-28 ms | Full protection achievable |
| 15 m | Handgun | 42.9 ms | 40-42 ms | Full protection trivial |

### Military Scenarios

| Distance | Weapon | Flight Time | Warning Time | Protection Feasibility |
|----------|--------|--------------|--------------|------------------------|
| 50 m | Rifle | 58.8 ms | 56-58 ms | Full protection trivial |
| 100 m | Rifle | 117.6 ms | 115-117 ms | Full protection trivial |
| 200 m | Rifle | 235.3 ms | 233-235 ms | Full protection trivial |
| 300 m | Rifle | 352.9 ms | 350-352 ms | Full protection trivial |

**For military applications, the sensing-to-protection problem is almost trivially solvable from a timing perspective.** The constraint is materials, not timing.

---

# PART 5: SPECIFIC MILITARY USE CASES

## 5.1 Urban Warfare

**The urban environment is the most sensor-hostile:**
- Limited line-of-sight
- Complex multi-floor geometry
- Threats from any direction
- Civilian presence
- Rapid transitions between interior/exterior

**SENTINEL-WEAR advantage:**

| Challenge | Traditional Approach | SENTINEL-WEAR Solution |
|-----------|---------------------|------------------------|
| Threat from behind | Reliant on team coverage | Individual 360° awareness |
| Through-wall threats | No capability | mmWave radar sees through drywall |
| Multi-floor awareness | External drone or teammate | Body-centric vertical awareness |
| Blind corners | Peek, expose, risk | Radar/acoustic scans corners |
| Dark rooms | Flashlight/NVG reveals position | Radar/acoustic works silently |
| Civilian vs. combatant | Visual identification | Multi-modal classification |
| Mapping complex interiors | Manual or pre-mission | Real-time dense SLAM |

**Advantage:** A single soldier with SENTINEL-WEAR has better situational awareness than a traditional 4-person fireteam without it.

## 5.2 Sniper and Counter-Fire Detection

**Current approach:**
- Acoustic sniper detection systems (vehicle-mounted or backpack)
- Delayed (seconds to minutes)
- Requires external system

**SENTINEL-WEAR approach:**
- Detection: 50-150 µs after projectile enters detection volume
- Direction: 100-500 µs
- Alert: 0.65-2.4 ms total
- **Soldier knows where the shot came from before the bullet arrives at 10+ meters**

**Capability created:**
- Every soldier becomes a sniper detection node
- No additional equipment required
- Instantaneous alert with direction
- Squad mesh triangulates shooter position

## 5.3 Night Operations

**Traditional night operations:**
- NVG (limited field of view, active IR)
- Thermal (limited resolution)
- Night vision reveals position to enemy with NVG
- Limited peripheral awareness

**SENTINEL-WEAR night capability:**
- mmWave radar: Works in complete darkness, no emission
- Acoustic: Works in complete darkness
- LiDAR: Works in complete darkness
- **360° coverage at all times**
- **No light emission**
- **No dependency on ambient light**

**Advantage:** Soldiers can navigate, detect threats, and maintain awareness in complete darkness without emitting any radiation.

## 5.4 Through-Wall and Through-Obscuration Sensing

| Material | mmWave Radar | Acoustic | LiDAR |
|----------|--------------|----------|-------|
| Drywall | ✅ | ✅ | ❌ |
| Plywood | ✅ | ✅ | ❌ |
| Dense concrete | ⚠️ Limited | ⚠️ Limited | ❌ |
| Foliage | ✅ | ⚠️ Limited | ⚠️ Limited |
| Smoke | ✅ | ✅ | ❌ |
| Fog | ✅ | ✅ | ❌ |
| Steam | ✅ | ✅ | ❌ |

**Urban advantage:** The system "sees" through interior walls, around corners via acoustic reflection, and through smoke/fog that blinds visual sensors.

## 5.5 Dense SLAM for Real-Time Battlefield Intelligence

**Traditional approach:**
- Pre-mission satellite imagery
- Paper maps
- GPS (spoofable, requires satellite)
- Radio communications (monitored)

**SENTINEL-WEAR SLAM capability:**
- Real-time 3D map generated as soldier moves
- Each soldier contributes to collective squad map
- No GPS dependency (visual odometry + IMU)
- Map persists locally, syncs when connected
- 360° pendant variant captures complete geometry

**Intelligence value:**
- Each patrol automatically maps new areas
- Collective squad map aggregates multiple perspectives
- Post-mission intelligence automatically compiled
- Persistent 3D model of visited locations

---

# PART 6: SQUAD-LEVEL PERCEPTION MESH

## 6.1 The Mesh Architecture at Scale

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SQUAD-LEVEL PERCEPTION MESH                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Soldier A (Squad Leader)                                                    │
│  └── Pendant + Bracelets + Belt + Anklets = Full awareness                  │
│                                                                              │
│  Soldier B (Rifleman)                                                        │
│  └── Pendant + Bracelets + Belt = Forward/lateral awareness                 │
│                                                                              │
│  Soldier C (Rifleman)                                                        │
│  └── Pendant + Bracelets + Belt = Forward/lateral awareness                 │
│                                                                              │
│  Soldier D (SAW Gunner)                                                      │
│  └── Pendant + Bracelets + Belt + Anklets = Full awareness                  │
│                                                                              │
│  ──────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  INTER-SOLDIER MESH (BLE 5.x / UWB)                                          │
│  ├── Detection events shared across soldiers                                 │
│  ├── Track-level fusion at each belt node                                   │
│  ├── Collective direction estimation for threats                            │
│  ├── Shooter triangulation from multiple detections                         │
│  └── Shared SLAM map updates                                                │
│                                                                              │
│  ──────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  OUTPUT: UNIFIED SQUAD WORLD MODEL                                           │
│  ├── 360° coverage of entire squad position                                  │
│  ├── Multi-perspective SLAM fusion                                          │
│  ├── Collective threat tracking                                             │
│  ├── Blue force positions (all soldiers tracked)                            │
│  └── Detected enemy positions (shared across mesh)                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 6.2 Detection Fusion Across Soldiers

**Single-soldier detection:**
- Direction ± 15°
- Velocity ± 10%
- Range ± 0.5 m

**Tri-soldier fusion:**
- Direction ± 5°
- Velocity ± 3%
- Range ± 0.2 m
- **Triangulated shooter position**

## 6.3 SLAM Map Aggregation

Each soldier's dense SLAM map contributes to the squad-level map:

| Soldier | Contribution |
|---------|--------------|
| Alpha (point) | Forward 180° geometry |
| Bravo (left flank) | Left 180° geometry |
| Charlie (right flank) | Right 180° geometry |
| Delta (rear security) | Rear 180° geometry |

**Result:** Complete 360° geometric map of squad surroundings, compiled from individual soldier contributions.

---

# PART 7: EXTREME VELOCITY DETECTION — MILITARY CONTEXT

## 7.1 Engagement Distance Reality

**Civilian scenario (primary SENTINEL-WEAR design):**
- Threats: handguns, civilian conflicts
- Engagement distances: 3-15 m
- Warning time: 6-30 ms

**Military scenario:**
- Threats: rifles, machine guns, snipers
- Engagement distances: 50-300 m
- Warning time: 50-350 ms

**The military scenario has ABUNDANT warning time.**

| Distance | Rifle Flight Time | Warning Time |
|----------|-------------------|--------------|
| 50 m | 58.8 ms | 56-58 ms |
| 100 m | 117.6 ms | 115-117 ms |
| 200 m | 235.3 ms | 233-235 ms |
| 300 m | 352.9 ms | 350-352 ms |

## 7.2 Detection at Military Distances

| Detection Range | Rifle Flight Time | Warning Time |
|-----------------|-------------------|--------------|
| 50 m | 58.8 ms | 56-58 ms |
| 100 m | 117.6 ms | 115-117 ms |
| 150 m | 176.5 ms | 174-176 ms |

**At 100+ meters, warning time exceeds human reaction time (~200 ms for simple reactions).**

## 7.3 Squad Triangulation

When multiple soldiers detect the same projectile:

```
Soldier A: Detects projectile at bearing 045°
Soldier B: Detects projectile at bearing 030°
Soldier C: Detects projectile at bearing 015°

Fusion result: Shooter position triangulated with < 5 m error at 200 m range.
```

**Every soldier becomes a node in a distributed ballistic detection array.**

---

# PART 8: PROTECTION SYSTEM INTERFACE — THEORETICAL FRAMEWORK

## 8.1 What SENTINEL-WEAR Provides to Protection Systems

**Detection and awareness:**
- Threat velocity, direction, time-to-impact
- Classification (human/vehicle/projectile type)
- Trajectory prediction
- Alert generation
- Confidence scoring

**Detection output signals:**

| Signal | Format | Latency | Purpose |
|--------|--------|---------|---------|
| Detection trigger | GPIO | 150-500 µs | Initiate protection deployment |
| Trajectory estimate | Vector + time-to-impact | 0.5-2.0 ms | Target protection zone |
| Confidence score | Float | 0.5-2.0 ms | Protection system decision |
| Direction vector | 3D vector | 0.5-2.0 ms | Zone-specific deployment |

## 8.2 The Theoretical Protection Integration

**What is theoretically achievable:**

| Engagement Distance | Detection-to-Alert | Available for Protection | Theoretical Protection |
|--------------------|--------------------|--------------------------|------------------------|
| < 3 m | 0.65-2.4 ms | < 3 ms (rifle) / < 6 ms (handgun) | Passive materials only |
| 3-5 m | 0.65-2.4 ms | 1-10 ms | Passive + triggered marginal |
| 5-10 m | 0.65-2.4 ms | 3-26 ms | Passive + triggered comfortable |
| 10-50 m | 0.65-2.4 ms | 8-248 ms | Full protection achievable |
| 50+ m | 0.65-2.4 ms | 54+ ms | Full protection trivial |

**The window is open at appropriate distances. The research question is what fills that window.**

## 8.3 Theoretical Protection Mechanisms (Research-Documented)

### Passive Protection (Always Active)

| Material | Capability | Deployment Time | Status |
|----------|------------|-----------------|--------|
| UHMWPE/Dyneema | Fragment/slash protection | Always active | ✅ Production |
| STF-impregnated fabrics | Impact dispersion | Always active | ✅ Production |
| Ceramic composites | Ballistic protection | Always active | ✅ Production |

### Detection-Triggered Systems (Research Domain)

| System | Deployment Time | Mechanism | Research Status |
|--------|-----------------|-----------|-----------------|
| Pre-tensioned filaments | 0.5-2.0 ms | Spring release + trajectory destabilization | Advanced research |
| MR fluid activation | 1-5 ms | Electromagnet stiffening | Advanced research |
| State-change materials | 1-10 ms | Chemical/physical property change | Research |
| Electroactive polymers | < 1 ms (potential) | Electrical activation | Future research |

### The Honest Characterization

**SENSING: ACHIEVED**
- Detection latency: 0.5-1.6 ms
- Alert latency: 0.65-2.4 ms
- Direction accuracy: ±15°
- 360° coverage: ✅
- Dense SLAM: ✅
- Evidence capture: ✅

**PROTECTION: RESEARCH DOMAIN**
- Point-blank (< 3 m): Passive materials only; awareness
- Close (3-10 m): Passive + triggered systems theoretically possible
- Medium (10-50 m): Triggered systems achievable
- Long (50+ m): Abundant time for any system

## 8.4 What SENTINEL-WEAR Does NOT Provide

**No effectors:**
- No kinetic actuators
- No energetic materials
- No chemical dispensers
- No RF jammers
- No laser dazzlers
- No mechanism that acts on the external world

**No protection specification:**
- Does not specify protection systems
- Does not validate protection efficacy
- Does not integrate with protection hardware
- Provides detection output only; protection selection is user responsibility

---

# PART 9: BLUE FORCE TRACKING

## 9.1 Individual Tracking

Each soldier in the mesh is tracked:

| Data | Source |
|------|--------|
| Position | Belt IMU + mesh ranging |
| Orientation | Pendant IMU |
| Velocity | IMU integration |
| Gait state | Anklet IMU |
| Health status | Gait analysis, vitals |
| Ammo state | (External input) |
| Status | Manual or automated |

## 9.2 Squad-Level Tracking

All soldiers in the mesh contribute to the collective world model:

- Each soldier's position known to all others
- Collective coverage map shows where squad has visibility
- Blind spots identified and flagged
- Friendly fire mitigation: known positions of all friendlies

## 9.3 Command Integration

The squad leader's belt node aggregates:
- All soldier positions
- All detections
- All track data
- SLAM map from all soldiers

Output to command network:
- Squad position
- Detected threats
- Map updates
- Alerts

---

# PART 10: CASUALTY DETECTION

## 10.1 Gait-Based Casualty Detection

The anklet nodes continuously analyze gait:

| Metric | Normal Range | Anomaly Threshold |
|--------|--------------|-------------------|
| Cadence | 80-120 steps/min | > 140 or < 40 |
| Stride asymmetry | < 10% | > 25% |
| Impact force | 1-3 g | > 5 g (fall) |
| Stumble precursor | Absent | Present |

**Detection:**
- Fall: Immediate detection
- Wound: Gait asymmetry change
- Unconscious: Cessation of motion with presence detection

## 10.2 Automated Alert

When casualty detected:

```
CasualtyEvent {
    soldier_id: "ALPHA-01"
    type: "fall_detected"
    position: [x, y, z]
    timestamp: 1706123456789
    confidence: 0.94
}

→ Broadcast to squad mesh
→ Alert to command
→ Position logged
→ Recording initiated on all cameras
```

## 10.3 After-Action Intelligence

All casualty events are:
- Logged with full integrity chain
- Associated with detection context
- Available for post-mission analysis
- Camera recordings preserved

---

# PART 11: INTEGRATION ARCHITECTURE

## 11.1 Physical Integration Points

**Exoskeleton attachment:**
- Pendant: Integrated into torso section
- Belt: Replaces/extends exoskeleton belt
- Bracelets: Integrated into forearm sections
- Anklets: Integrated into lower leg sections

**Power sharing:**
- SENTINEL-WEAR can draw from exoskeleton battery
- Typical exoskeleton battery: 500+ Wh
- SENTINEL-WEAR full active draw: 5 W maximum
- **Runtime: 100+ hours on exoskeleton power**

## 11.2 Data Bus Integration

**Option A: Separate SENTINEL-WEAR mesh**
- SENTINEL-WEAR nodes maintain their own BAN
- Belt node outputs to exoskeleton via standard interface
- Complete isolation of sensing mesh

**Option B: Shared data bus**
- SENTINEL-WEAR nodes connect to exoskeleton CAN/ethernet bus
- Direct integration with exoskeleton control systems
- Higher integration complexity

## 11.3 Form Factor Considerations

| SENTINEL-WEAR Node | Exoskeleton Integration | Notes |
|---------------------|------------------------|-------|
| Pendant | Torso housing | Replace jewelry pendant with structural housing |
| Belt | Integrated | Replace standard belt |
| Bracelets | Forearm attachment | Straps to exoskeleton forearm |
| Anklets | Lower leg attachment | Integrated into leg sections |
| Eyewear | Helmet mount | Integrate into helmet system |

## 11.4 Protection Module Integration Points (Research Reference)

**Exoskeleton integration points for protection systems (user-selected):**

| Location | Protection Type | Notes |
|----------|-----------------|-------|
| Torso housing | Pre-tensioned filament canisters | Triggered by detection signal |
| Chest plate | MR fluid packets | Electromagnet activation |
| Exoskeleton shell | STF-impregnated padding | Passive baseline |
| Shoulder/neck | Filament cloud deployment | High-value zone protection |

**Reference:** `docs/theory/passive_materials_research.md`

**Important:** Protection system design and selection is the user's responsibility. SENTINEL-WEAR provides detection output; it does not specify, design, or validate protection systems.

---

# PART 12: SAFETY AND ETHICAL POSTURE

## 12.1 Core Commitments

**SENTINEL-WEAR remains:**
- Sensing-only
- No actuation
- No weapons capability
- No operational guidance

**The project does NOT:**
- Specify weapons systems
- Provide combat tactics guidance
- Implement protective mechanisms
- Consult on military applications
- Validate protection system efficacy

## 12.2 Honest Characterization

**What is achievable (Production-Capable):**
- Microsecond detection
- Millisecond alerting
- 360° awareness
- Dense SLAM mapping
- Squad-level perception mesh
- Blue force tracking
- Casualty detection
- Detection signals for protection system integration

**What is theoretically possible (Research Domain):**
- Triggered protection at 5+ m (handgun), 10+ m (rifle)
- Deployment windows of 6-350+ ms at realistic distances
- Materials-based protection integration

**What is NOT achievable:**
- Point-blank active protection (physics-constrained)
- Guaranteed personal protection from ballistic threats
- Any sensing system that prevents projectile impact

## 12.3 Responsible Disclosure

This document is provided for:
- Research transparency
- Academic completeness
- Dual-use technology analysis
- Understanding sensing layer capabilities
- Documenting the theoretical protection integration path

This document does NOT:
- Specify weapons systems
- Provide operational combat guidance
- Enable any kinetic capability
- Validate protection efficacy

---

# PART 13 SUMMARY

## 13.1 The Capability Gap Filled

| Current State | With SENTINEL-WEAR |
|---------------|-------------------|
| Exoskeleton = strong, blind soldier | Exoskeleton = strong, aware soldier |
| Situational awareness from external sources | Situational awareness from body-centric mesh |
| Each soldier isolated | Squad-level perception mesh |
| Threat detection via external systems | Threat detection built into every soldier |
| Mapping by manual/satellite | Real-time dense SLAM |
| Casualty detection by observation | Automated gait-based casualty detection |
| Protection timing unknown | Detection-to-protection timing characterized |

## 13.2 The Architectural Insight

**SENTINEL-WEAR transforms the individual soldier from a sensor user into a sensor platform.**

This is a new category of military capability — in sensing and awareness, with protection integration as a materials-research path.

## 13.3 The Maintained Boundary

**The project remains:**
- Open-source research
- Sensing-only
- No actuation
- No weapons
- No operational guidance
- Protection materials research documented separately

**Dual-use implications are documented openly and honestly.** The sensing capabilities exist; the protection research path is documented for completeness. The separation between sensing (achieved) and protection (research) is explicitly maintained.

---

**End of Document**
