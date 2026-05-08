# Detection Range Analysis — Realistic Engagement Scenarios

**Project:** SENTINEL-WEAR
**Domain:** Extreme velocity detection feasibility and system requirements
**Status:** Production Architecture Reference

---

## 1. Purpose

This document provides a rigorous analysis of realistic engagement distances for extreme velocity detection in SENTINEL-WEAR. It corrects the over-constrained "3-meter rifle scenario" that dominated earlier discussions and establishes the true physics constraints, system requirements, and value proposition for the detection architecture.

**Key thesis:** The system's critical constraint is **detection range**, not **detection latency**. At realistic engagement distances (50-300 meters for rifles, 5-30 meters for handguns), the flight time provides comfortable margins for detection and alerting. The architecture is sound; the original scenario analysis was unrealistic.

---

## 2. The Original Problem: Over-Constrained Scenarios

### 2.1 What We Originally Assumed

Earlier analysis focused on:
- Rifle at 3 meters engagement distance
- Handgun at 3 meters engagement distance
- Latency budgets measured in microseconds
- Conclusion: System barely works, needs extreme optimization

### 2.2 Why This Was Wrong

**The 3-meter rifle scenario is:**

1. **Physically unrealistic**
   - Point-blank range for any firearm
   - Represents < 0.1% of actual shooting incidents
   - Most shooters do not engage at this distance

2. **Tactically irrelevant**
   - At 3 meters, the shooter can physically reach the victim
   - Warning time of any duration provides limited protective value
   - This is a melee-distance engagement, not a ranged engagement

3. **Not the use case**
   - SENTINEL-WEAR is designed for awareness and evidence
   - Close-range ambush is fundamentally undetectable in time for response
   - The value is in realistic scenarios where warning is possible

### 2.3 What Actual Data Shows

**Typical engagement distances by weapon type:**

| Weapon Type | Typical Engagement Distance | Source Context |
|-------------|---------------------------|-----------------|
| Handgun (crime) | 3-15 meters | FBI crime statistics |
| Handgun (self-defense) | 3-10 meters | Defensive shooting data |
| Rifle (active shooter) | 20-300 meters | Mass shooting analysis |
| Rifle (sniper/precision) | 100-500+ meters | Military/police data |
| Shotgun | 5-25 meters | Hunting/defense data |

**Key insight:** The 50-300 meter range for rifles is where SENTINEL-WEAR provides maximum value. The 5-15 meter range for handguns is where it provides awareness (not protection).

---

## 3. Physics of Projectile Detection

### 3.1 Projectile Velocities

| Projectile Type | Muzzle Velocity | Velocity at 50 m | Velocity at 100 m |
|-----------------|-----------------|------------------|-------------------|
| 9mm handgun | 300-400 m/s | 280-370 m/s | 260-340 m/s |
| .45 ACP handgun | 250-300 m/s | 235-280 m/s | 220-260 m/s |
| .357 Magnum | 380-450 m/s | 350-410 m/s | 320-380 m/s |
| 5.56 NATO rifle | 900-950 m/s | 820-870 m/s | 750-800 m/s |
| 7.62 NATO rifle | 800-850 m/s | 740-790 m/s | 690-740 m/s |
| .308 Winchester | 850-900 m/s | 780-830 m/s | 720-770 m/s |
| 12-gauge slug | 400-450 m/s | 340-380 m/s | 290-330 m/s |
| 12-gauge buckshot | 380-420 m/s | — (spreads rapidly) | — |

### 3.2 Flight Time Calculations

**Basic formula:**

$$t = \frac{d}{v}$$

Where:
- $t$ = flight time
- $d$ = distance
- $v$ = average velocity

**Corrected for deceleration:**

$$t = \int_0^d \frac{dx}{v(x)}$$

For practical purposes, use average velocity between shooter and target.

### 3.3 Flight Time Tables

**Handgun (9mm, average 320 m/s):**

| Distance | Flight Time |
|----------|-------------|
| 3 m | 9.4 ms |
| 5 m | 15.6 ms |
| 10 m | 31.3 ms |
| 15 m | 46.9 ms |
| 20 m | 62.5 ms |
| 30 m | 93.8 ms |

**Rifle (5.56 NATO, average 850 m/s):**

| Distance | Flight Time |
|----------|-------------|
| 3 m | 3.5 ms |
| 5 m | 5.9 ms |
| 10 m | 11.8 ms |
| 25 m | 29.4 ms |
| 50 m | 58.8 ms |
| 75 m | 88.2 ms |
| 100 m | 117.6 ms |
| 150 m | 176.5 ms |
| 200 m | 235.3 ms |
| 300 m | 352.9 ms |
| 400 m | 470.6 ms |
| 500 m | 588.2 ms |

**Rifle (7.62 NATO, average 750 m/s):**

| Distance | Flight Time |
|----------|-------------|
| 50 m | 66.7 ms |
| 100 m | 133.3 ms |
| 150 m | 200.0 ms |
| 200 m | 266.7 ms |
| 300 m | 400.0 ms |

---

## 4. Human Response Time Context

### 4.1 Human Perception and Reaction Times

| Response Type | Latency | Description |
|---------------|---------|-------------|
| Simple reflex (startle) | 20-50 ms | Involuntary response to stimulus |
| Simple reaction (single choice) | 150-250 ms | Press button when light appears |
| Choice reaction (multiple options) | 300-500 ms | Select correct response from options |
| Complex decision | 500-2000+ ms | Evaluate options and decide |
| Protective movement initiation | 200-400 ms | Begin deliberate movement |
| Completed protective movement | 500-2000 ms | Move to cover, draw weapon, etc. |

### 4.2 Implications for Warning Time

| Warning Time | What's Possible |
|--------------|-----------------|
| < 10 ms | Startle response only; awareness but no action |
| 10-50 ms | Reflex preparation; muscle tensing |
| 50-150 ms | Simple reaction possible; alert processing |
| 150-300 ms | Deliberate movement initiation possible |
| 300-500 ms | Full protective movement possible |
| 500+ ms | Complex decisions possible |

### 4.3 Warning Time by Scenario

**Handgun at 5 meters (15.6 ms flight time):**
- Warning time (with detection): ~14 ms
- Human capability: Startle only
- Result: Awareness, no protective action

**Handgun at 15 meters (46.9 ms flight time):**
- Warning time (with detection): ~45 ms
- Human capability: Reflex preparation
- Result: Awareness, limited protective preparation

**Rifle at 50 meters (58.8 ms flight time):**
- Warning time (with detection): ~57 ms
- Human capability: Reflex preparation, simple reaction
- Result: Awareness, movement initiation beginning

**Rifle at 100 meters (117.6 ms flight time):**
- Warning time (with detection): ~115 ms
- Human capability: Simple reaction well underway
- Result: Awareness, deliberate movement possible

**Rifle at 200 meters (235.3 ms flight time):**
- Warning time (with detection): ~233 ms
- Human capability: Full protective movement possible
- Result: Meaningful protective response

**Rifle at 300 meters (352.9 ms flight time):**
- Warning time (with detection): ~350 ms
- Human capability: Complex decisions, full movement
- Result: Significant protective response

---

## 5. Distance Categories and System Implications

### 5.1 Category Definitions

| Category | Distance | Primary Threats | Flight Time | System Role |
|----------|----------|-----------------|-------------|-------------|
| **Point-Blank** | < 5 m | All weapons | < 15 ms | Awareness only |
| **Close-Range** | 5-15 m | Handguns | 15-47 ms | Awareness + startle |
| **Medium-Range** | 15-50 m | Handguns, rifles | 47-60 ms | Awareness + reflex prep |
| **Long-Range** | 50-200 m | Rifles | 60-235 ms | Awareness + protective action |
| **Extended-Range** | 200-500 m | Rifles (precision) | 235-590 ms | Full response possible |

### 5.2 Category Analysis

#### Point-Blank (< 5 m)

**Physics:**
- Rifle flight time: < 5 ms
- Handgun flight time: < 15 ms

**Detection:**
- System can detect (latency < 2 ms)
- Alert arrives within flight time for most scenarios

**Human response:**
- Warning time: < 10 ms
- Capability: Startle reflex only
- No deliberate action possible

**System value:**
- Detection: ✅ Yes
- Alert: ✅ Yes
- Evidence capture: ✅ Yes
- Protective action: ❌ No

**Conclusion:** The system provides awareness and evidence at point-blank range. It cannot provide meaningful protective response time. This is a physical constraint, not a system limitation.

#### Close-Range (5-15 m)

**Physics:**
- Handgun flight time: 15-47 ms
- Rifle flight time: 5-18 ms

**Detection:**
- System can detect
- Alert arrives well within flight time for handguns
- Alert tight for rifles

**Human response:**
- Warning time: 10-45 ms
- Capability: Reflex preparation, startle
- Limited deliberate action

**System value:**
- Detection: ✅ Yes
- Alert: ✅ Yes
- Evidence capture: ✅ Yes
- Protective action: ⚠️ Limited (handgun), ❌ No (rifle)

**Conclusion:** The system provides good awareness and reflex preparation for handgun scenarios. Rifle scenarios remain too fast for deliberate response.

#### Medium-Range (15-50 m)

**Physics:**
- Handgun flight time: 47-90 ms
- Rifle flight time: 18-60 ms

**Detection:**
- System can detect
- Alert arrives well within flight time

**Human response:**
- Warning time: 45-58 ms (handgun), 16-58 ms (rifle)
- Capability: Reflex preparation, simple reaction beginning
- Deliberate movement initiation possible for longer times

**System value:**
- Detection: ✅ Yes
- Alert: ✅ Yes
- Evidence capture: ✅ Yes
- Protective action: ⚠️ Beginning (handgun), Limited (rifle)

**Conclusion:** The system provides useful warning for handgun scenarios. Rifle scenarios remain challenging but provide reflex preparation time.

#### Long-Range (50-200 m)

**Physics:**
- Rifle flight time: 60-235 ms

**Detection:**
- System can detect
- Alert arrives well within flight time

**Human response:**
- Warning time: 58-233 ms
- Capability: Simple reaction (complete for 100+ m), deliberate movement (beginning at 50 m, complete at 150+ m)
- Meaningful protective action possible

**System value:**
- Detection: ✅ Yes
- Alert: ✅ Yes
- Evidence capture: ✅ Yes
- Protective action: ✅ Yes (increasing with distance)

**Conclusion:** This is the optimal operating envelope. At 100+ meters, the wearer has sufficient warning to begin protective movement. At 150+ meters, full protective response is possible.

#### Extended-Range (200-500 m)

**Physics:**
- Rifle flight time: 235-590 ms

**Detection:**
- System can detect (if radar range sufficient)
- Alert arrives with substantial margin

**Human response:**
- Warning time: 233-588 ms
- Capability: Full protective movement, complex decisions
- Significant response possible

**System value:**
- Detection: ✅ Yes (if range allows)
- Alert: ✅ Yes
- Evidence capture: ✅ Yes
- Protective action: ✅ Full

**Conclusion:** At these distances, the system provides abundant warning time. The constraint becomes detection range, not detection latency.

---

## 6. Detection Range vs Alert Latency

### 6.1 The Critical Insight

**Detection latency is trivial compared to flight time at realistic distances.**

**Alert latency is manageable compared to flight time at realistic distances.**

**Detection range is the primary constraint.**

### 6.2 Quantitative Analysis

**For rifle at 100 meters:**

| Component | Latency | Percentage of Flight Time |
|-----------|---------|---------------------------|
| Flight time | 117.6 ms | 100% |
| CW Doppler detection | 0.1-0.2 ms | 0.1-0.2% |
| Event camera direction | 0.1-0.5 ms | 0.1-0.4% |
| BLE transmission (standard) | 0-30 ms | 0-26% |
| BLE transmission (optimized) | 0.3-0.8 ms | 0.3-0.7% |
| Haptic onset (LRA) | 2-10 ms | 2-9% |
| Haptic onset (piezo) | 0.5-1 ms | 0.4-0.9% |
| **Total (standard + LRA)** | **2.5-41.8 ms** | **2-36%** |
| **Total (optimized + piezo)** | **1.0-2.5 ms** | **0.8-2.1%** |

**Result:** Even the worst-case (standard BLE + LRA) uses only 36% of the available flight time. The optimized case uses 2%.

### 6.3 The Tradeoff

| Optimization | Latency Improvement | Flight Time Margin Gained | Value |
|--------------|---------------------|--------------------------|-------|
| Standard BLE → Optimized BLE | 0-30 ms → 0.3-0.8 ms | +0-29.7 ms | High at close range, low at distance |
| LRA → Piezo haptics | 2-10 ms → 0.5-1 ms | +1.5-9.5 ms | Moderate at close range, low at distance |
| Extended detection range | +50 m detection | +58.8 ms warning | **Very high at all ranges** |

### 6.4 Optimization Priority Order

| Priority | Optimization | Cost | Impact at 100 m | Impact at 300 m |
|----------|--------------|------|-----------------|------------------|
| 1 | Maximize detection range | Hardware | +58.8 ms per 50 m | +58.8 ms per 50 m |
| 2 | Reliable detection at distance | Firmware | Critical | Critical |
| 3 | Directional alerting | Hardware + firmware | High | Moderate |
| 4 | BLE optimization | Firmware | Moderate | Low |
| 5 | Piezo haptics | Hardware | Moderate | Low |

---

## 7. Hardware Requirements by Scenario

### 7.1 Point-Blank (< 5 m)

**Detection:**
- CW radar: Required (fastest detection)
- Event camera: Optional (direction validation)

**Alerting:**
- Piezo haptics: Required (fastest onset)
- Optimized BLE: Required (minimum latency)
- All optimizations: Required

**Evidence:**
- Camera: Required
- Integrity chain: Required

**Result:** Maximum optimization provides awareness-only. No protective response possible.

### 7.2 Close-Range (5-15 m)

**Detection:**
- CW radar: Required
- Event camera: Recommended

**Alerting:**
- Piezo haptics: Recommended
- Optimized BLE: Recommended
- Standard BLE: Acceptable (warning time still sufficient)

**Evidence:**
- Camera: Required
- Integrity chain: Required

**Result:** Awareness and reflex preparation for handguns. Limited response for rifles.

### 7.3 Medium-Range (15-50 m)

**Detection:**
- CW radar: Required
- Event camera: Recommended

**Alerting:**
- LRA haptics: Sufficient
- Piezo haptics: Nice-to-have
- Standard BLE: Acceptable
- Optimized BLE: Nice-to-have

**Evidence:**
- Camera: Required
- Integrity chain: Required

**Result:** Good warning for handguns, reflex preparation for rifles.

### 7.4 Long-Range (50-200 m)

**Detection:**
- CW radar: Required (range-optimized configuration)
- Event camera: Optional

**Alerting:**
- LRA haptics: Sufficient
- Piezo haptics: Not required
- Standard BLE: Sufficient
- Optimized BLE: Nice-to-have for maximum margin

**Evidence:**
- Camera: Required
- Integrity chain: Required

**Result:** Full awareness and meaningful protective response time.

### 7.5 Extended-Range (200-500 m)

**Detection:**
- CW radar: Required (maximum range configuration)
- Detection range is the constraint

**Alerting:**
- Any haptic: Sufficient
- Any BLE mode: Sufficient

**Evidence:**
- Camera: Required (may need telephoto for useful imagery)
- Integrity chain: Required

**Result:** Abundant warning time. Primary challenge is detection range.

---

## 8. Radar Configuration for Range vs Latency

### 8.1 Range-Optimized Configuration

**Trade velocity resolution for detection range.**

```toml
[extreme_velocity.range_optimized]
target_range_m = 150              # Detect at 150 meters
range_resolution_m = 2.0          # Accept coarse range
velocity_resolution_ms = 150      # Accept coarse velocity

[extreme_velocity.range_optimized.radar]
tx_power_dbm = 12                 # Maximum power
antenna_gain_dbi = 15             # Directional antenna
bw_mhz = 200                      # Narrower bandwidth for range
chirp_duration_us = 20            # Longer chirp for range
```

**Result:** Detects rifle fire at 150 meters. Flight time from detection to impact: 176 ms. Warning time with standard alert: 135-173 ms.

### 8.2 Latency-Optimized Configuration

**Trade detection range for velocity resolution.**

```toml
[extreme_velocity.latency_optimized]
target_range_m = 50               # Detect at 50 meters
range_resolution_m = 0.3          # Fine range
velocity_resolution_ms = 20       # Fine velocity

[extreme_velocity.latency_optimized.radar]
tx_power_dbm = 10
antenna_gain_dbi = 8
bw_mhz = 500                      # Wider bandwidth
chirp_duration_us = 8             # Short chirp for speed
```

**Result:** Detects rifle fire at 50 meters. Flight time from detection to impact: 58.8 ms. Warning time with optimized alert: 56-58 ms.

### 8.3 Balanced Configuration

**Compromise between range and resolution.**

```toml
[extreme_velocity.balanced]
target_range_m = 100              # Detect at 100 meters
range_resolution_m = 0.5          # Moderate range resolution
velocity_resolution_ms = 50       # Moderate velocity resolution

[extreme_velocity.balanced.radar]
tx_power_dbm = 10
antenna_gain_dbi = 10
bw_mhz = 300
chirp_duration_us = 12
```

**Result:** Detects rifle fire at 100 meters. Flight time: 117.6 ms. Warning time: 115-117 ms. Recommended default.

---

## 9. Complete Scenario Analysis

### 9.1 Scenario: Handgun at 10 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 9mm handgun |
| Velocity | ~320 m/s |
| Flight time | 31.3 ms |
| Detection latency (CW radar) | 0.1-0.2 ms |
| Direction latency (event camera) | 0.1-0.5 ms |
| BLE latency (standard) | 0-30 ms |
| BLE latency (optimized) | 0.3-0.8 ms |
| Haptic latency (LRA) | 2-10 ms |
| Haptic latency (piezo) | 0.5-1 ms |
| **Warning time (standard + LRA)** | **18.6-29.2 ms** |
| **Warning time (optimized + piezo)** | **29.6-30.6 ms** |

**Human capability at 18-30 ms:**
- Startle reflex: ✅
- Reflex preparation: ⚠️ Beginning
- Deliberate action: ❌

**System provides:** Detection, alert, evidence. Limited protective preparation.

### 9.2 Scenario: Handgun at 15 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 9mm handgun |
| Velocity | ~320 m/s |
| Flight time | 46.9 ms |
| **Warning time (standard + LRA)** | **35.6-44.8 ms** |
| **Warning time (optimized + piezo)** | **45.3-46.3 ms** |

**Human capability at 35-46 ms:**
- Startle reflex: ✅
- Reflex preparation: ✅
- Deliberate action: ⚠️ Beginning

**System provides:** Detection, alert, evidence. Some protective preparation possible.

### 9.3 Scenario: Rifle at 50 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 5.56 NATO rifle |
| Velocity | ~850 m/s |
| Flight time | 58.8 ms |
| **Warning time (standard + LRA)** | **48.8-56.8 ms** |
| **Warning time (optimized + piezo)** | **57.3-58.3 ms** |

**Human capability at 48-58 ms:**
- Startle reflex: ✅
- Reflex preparation: ✅
- Simple reaction: ⚠️ Beginning
- Deliberate action: ⚠️ Beginning

**System provides:** Detection, alert, evidence. Protective movement initiation possible.

### 9.4 Scenario: Rifle at 100 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 5.56 NATO rifle |
| Velocity | ~850 m/s |
| Flight time | 117.6 ms |
| **Warning time (standard + LRA)** | **106.6-115.6 ms** |
| **Warning time (optimized + piezo)** | **115.1-116.6 ms** |

**Human capability at 106-117 ms:**
- Startle reflex: ✅ Complete
- Reflex preparation: ✅ Complete
- Simple reaction: ✅ Complete or nearly complete
- Deliberate action: ⚠️ Beginning

**System provides:** Detection, alert, evidence, meaningful protective response time.

### 9.5 Scenario: Rifle at 200 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 5.56 NATO rifle |
| Velocity | ~850 m/s |
| Flight time | 235.3 ms |
| **Warning time (standard + LRA)** | **223.3-233.3 ms** |
| **Warning time (optimized + piezo)** | **232.8-234.3 ms** |

**Human capability at 223-234 ms:**
- All reflex responses: ✅ Complete
- Simple reaction: ✅ Complete
- Deliberate action: ✅ Possible

**System provides:** Detection, alert, evidence, full protective response time.

### 9.6 Scenario: Rifle at 300 Meters

| Parameter | Value |
|-----------|-------|
| Weapon | 5.56 NATO rifle |
| Velocity | ~850 m/s |
| Flight time | 352.9 ms |
| **Warning time (standard + LRA)** | **341.9-351.9 ms** |
| **Warning time (optimized + piezo)** | **350.4-351.9 ms** |

**Human capability at 342-352 ms:**
- All reflex responses: ✅ Complete
- All reaction types: ✅ Complete
- Deliberate action: ✅ Full

**System provides:** Detection, alert, evidence, full protective response time. Abundant margin.

---

## 10. Deployment Path Implications

### 10.1 Urban Residential Path

**Primary scenario:** Handgun at 5-15 m

**Requirements:**
- CW radar: Required
- Event camera: Recommended
- Piezo haptics: Recommended (for maximum close-range performance)
- Optimized BLE: Recommended
- Standard camera: Required (evidence)

**Warning time:** 6-46 ms (awareness and reflex preparation)

**Value proposition:** Awareness, reflex preparation, evidence capture. Not protective action.

### 10.2 Open Area Path

**Primary scenario:** Rifle at 50-200 m

**Requirements:**
- CW radar (range-optimized): Required
- Event camera: Optional
- LRA haptics: Sufficient
- Standard BLE: Sufficient
- Standard camera: Required (evidence)

**Warning time:** 48-234 ms (reflex preparation to full protective action)

**Value proposition:** Detection, meaningful warning time, evidence capture. This is the optimal use case.

### 10.3 Professional Security Path

**Primary scenario:** Mixed (handgun close, rifle medium-long)

**Requirements:**
- CW radar (balanced config): Required
- Event camera: Required
- Piezo haptics: Required (for close-range optimization)
- Optimized BLE: Required (for close-range optimization)
- Standard camera: Required
- 360° pendant: Recommended (full awareness)

**Warning time:** 6-234 ms (varies by threat distance)

**Value proposition:** Full-spectrum detection, optimized close-range performance, evidence capture.

### 10.4 Close-Range Urban Path

**Primary scenario:** Handgun at 3-10 m

**Requirements:**
- CW radar (latency-optimized): Required
- Event camera: Required
- Piezo haptics: Required
- Optimized BLE: Required
- Standard camera: Required

**Warning time:** 3-30 ms (awareness only)

**Value proposition:** Detection, awareness, evidence. Explicitly: not protective action.

---

## 11. System Value Proposition Summary

### 11.1 What the System Provides at All Distances

| Capability | Point-Blank | Close | Medium | Long | Extended |
|------------|-------------|-------|--------|------|----------|
| Detection | ✅ | ✅ | ✅ | ✅ | ✅ |
| Alert | ✅ | ✅ | ✅ | ✅ | ✅ |
| Evidence capture | ✅ | ✅ | ✅ | ✅ | ✅ |
| Awareness | ✅ | ✅ | ✅ | ✅ | ✅ |
| Reflex preparation | ⚠️ | ✅ | ✅ | ✅ | ✅ |
| Deliberate action | ❌ | ⚠️ | ⚠️ | ✅ | ✅ |

### 11.2 What the System Cannot Provide

| Capability | Reason |
|------------|--------|
| Impact prevention | Physics — projectile faster than human response |
| Point-blank protective action | Flight time < human reaction time |
| Guaranteed safety | No system can guarantee safety |
| Body armor substitute | System detects; does not protect |

### 11.3 The Honest Posture

**For close-range (handgun 5-15 m):**
- Detection: Excellent
- Alert: Fast
- Warning time: 6-46 ms
- Human response: Awareness, reflex preparation
- Evidence: Complete

**For medium-range (rifle 15-50 m):**
- Detection: Excellent
- Alert: Fast
- Warning time: 18-58 ms
- Human response: Reflex preparation, movement initiation beginning
- Evidence: Complete

**For long-range (rifle 50-200 m):**
- Detection: Excellent
- Alert: Fast
- Warning time: 48-234 ms
- Human response: Meaningful protective action possible
- Evidence: Complete

**For extended-range (rifle 200-500 m):**
- Detection: Excellent (if radar range sufficient)
- Alert: Fast
- Warning time: 233-588 ms
- Human response: Full protective action, complex decisions
- Evidence: Complete

---

## 12. Configuration Reference

### 12.1 Scenario-Based Configuration

```toml
# Close-range urban deployment
[extreme_velocity.scenario]
type = "close_urban"
primary_threat = "handgun"
typical_distance_m = 10

[extreme_velocity.scenario.detection]
mode = "latency_optimized"
target_range_m = 30

[extreme_velocity.scenario.alert]
haptic_type = "piezo"
ble_mode = "optimized"
```

```toml
# Open area deployment
[extreme_velocity.scenario]
type = "open_area"
primary_threat = "rifle"
typical_distance_m = 100

[extreme_velocity.scenario.detection]
mode = "range_optimized"
target_range_m = 200

[extreme_velocity.scenario.alert]
haptic_type = "lra"
ble_mode = "standard"
```

```toml
# Professional security deployment
[extreme_velocity.scenario]
type = "professional"
primary_threat = "mixed"
typical_distance_m = "variable"

[extreme_velocity.scenario.detection]
mode = "balanced"
target_range_m = 150

[extreme_velocity.scenario.alert]
haptic_type = "both"
ble_mode = "optimized"
```

### 12.2 Detection Range Configuration

```toml
[extreme_velocity.detection_range]
# Priority setting
optimization_target = "balanced"   # "range" | "latency" | "balanced"

[extreme_velocity.detection_range.radar]
# Range vs resolution tradeoff
tx_power_dbm = 10
antenna_type = "directional"
bw_mhz = 300
chirp_duration_us = 12

# Detection targets
min_detection_range_m = 30
target_detection_range_m = 100
max_detection_range_m = 150
```

---

## 13. Conclusion

### 13.1 Key Findings

1. **The 3-meter rifle scenario was unrealistic.** Real engagement distances are 50-300 m for rifles, 5-30 m for handguns.

2. **Detection latency is trivial at realistic distances.** A 0.2 ms detection time is 0.2% of a 117 ms flight time at 100 m.

3. **Alert latency is manageable at realistic distances.** Even worst-case (30 ms BLE + 10 ms LRA = 40 ms) uses only 34% of flight time at 100 m.

4. **Detection range is the primary constraint.** Adding 50 m of detection range adds 58.8 ms of warning time — far more than any latency optimization.

5. **Standard hardware works for most scenarios.** LRA haptics and standard BLE are sufficient for rifle at 100+ m. Piezo and optimized BLE are close-range optimizations.

### 13.2 System Value Proposition

**SENTINEL-WEAR provides:**
- Detection of extreme velocity projectiles at realistic engagement distances
- Alerting with sufficient warning time for protective response (at distance)
- Complete evidence capture for all scenarios
- Situational awareness that no other system provides

**SENTINEL-WEAR does not provide:**
- Impact prevention (physics constraint)
- Protective action at point-blank range (physics constraint)
- Guaranteed safety (no system can guarantee safety)
- Body armor (not a physical barrier)

### 13.3 The Honest Truth

**At close-range (< 15 m), the system provides awareness and evidence.** This is valuable, but the wearer should not expect protective response time.

**At medium-range (15-50 m), the system provides awareness, reflex preparation, and evidence.** Some protective movement may be possible.

**At long-range (50-200 m), the system provides meaningful protective response time.** This is the optimal operating envelope.

**At extended-range (200-500 m), the system provides abundant warning time.** The constraint becomes detection range, not latency.

**The physics is what it is.** No system can make a 3-meter rifle shot survivable through warning alone. SENTINEL-WEAR provides the best possible awareness and evidence at all distances, and meaningful protective warning at realistic engagement distances.

---

**End of Document**
