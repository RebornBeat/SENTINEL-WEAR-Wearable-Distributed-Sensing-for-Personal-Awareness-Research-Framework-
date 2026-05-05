# Hardware Configuration — SENTINEL-WEAR

**Version:** 0.1 | **Status:** Research Reference Design

---

## 1. Philosophy

This document specifies the hardware interfaces, power architecture, and configuration standards for SENTINEL-WEAR sensing nodes. It is a **reference design**, not a constraint specification.

**Core Principles:**
- **No Artificial Limits:** Interfaces support standard protocols at their maximum defined speeds. We do not limit resolution, range, or storage.
- **Modularity:** Each functional block (Sensing, Compute, Radio, Power) is defined by its interface, not a specific part number. Substitutions are expected as technology evolves.
- **Connectivity Agnostic:** The hardware supports any radio standard compatible with the MCU interface (BLE, UWB, Custom ISM).
- **Storage Agnostic:** Designs support local storage (Flash, SD) or streaming to belt controller.

---

## 2. Core Node Architecture

Every SENTINEL-WEAR node shares a common architectural baseline. Variants differ only in which sensor interfaces are populated.

```
┌─────────────────────────────────────────────────────────────┐
│                       Node Architecture                      │
├─────────────────────────────────────────────────────────────┤
│  MCU Core                                                   │
│  - ARM Cortex-M4F / RISC-V (configurable)                   │
│  - Floating Point Unit (FPU) required for IMU fusion        │
│  - DSP extensions preferred for radar/audio processing      │
│  - Security element (Secure Boot, Crypto acceleration)      │
├─────────────────────────────────────────────────────────────┤
│  Sensing Interfaces (Populated per Node Type)                │
│  - mmWave Radar Interface (SPI/UART)                        │
│  - IMU Interface (I2C/SPI)                                  │
│  - Acoustic Interface (I2S/PDM)                             │
│  - LiDAR/ToF Interface (UART/SPI)                           │
│  - Environmental Interface (I2C)                            │
│  - Event Sensor Interface (MIPI/SPI)                        │
│  - Identification Module Interface (MIPI/USB/CSI)           │
├─────────────────────────────────────────────────────────────┤
│  Body-Area Network (BAN) Radio                              │
│  - Interface: UART / SPI / SDIO                             │
│  - Protocol Stack: Software-defined (BLE 5.x / UWB / Proprietary) │
│  - Antenna: PCB trace or ceramic chip antenna               │
├─────────────────────────────────────────────────────────────┤
│  Power Management Unit (PMU)                                │
│  - Battery Charger (Linear/Switching)                       │
│  - Fuel Gauge Interface (I2C)                               │
│  - Buck/Boost Regulators                                    │
│  - Power Path Management                                    │
├─────────────────────────────────────────────────────────────┤
│  User Interface (Optional)                                   │
│  - Haptic Driver (GPIO + H-Bridge)                          │
│  - Status LEDs (GPIO)                                       │
│  - Hardware Buttons (Reset, User, Kill Switch)              │
├─────────────────────────────────────────────────────────────┤
│  Debug Interface                                            │
│  - SWD (Serial Wire Debug)                                  │
│  - UART Console                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Power Architecture

### 3.1 Power Source
**Battery Chemistry:** Li-Po (Lithium Polymer) or Li-Ion cylindrical.
**Voltage:** 3.7V – 4.2V nominal.
**Capacity:** **Unconstrained.** Design supports capacities from 100mAh (miniature nodes) to 5000mAh+ (belt controller). The PMU must support the full range of charge currents.

### 3.2 Power Rails
| Rail | Voltage | Purpose | Notes |
|---|---|---|---|
| `VBAT` | 3.7V – 4.2V | Direct battery power | For radio PA, haptic driver |
| `VCC_3V3` | 3.3V | Main digital rail | MCU, Sensors, Radio |
| `VCC_1V8` | 1.8V | Low-power peripherals | IMU, Environmental sensors |
| `VCC_5V` | 5.0V | High-power sensors | Some LiDAR/ToF modules |

### 3.3 Charging
**Interface:** USB-C (USB 2.0) or Qi wireless charging coil.
**Standard:** USB-C PD (Power Delivery) optional.
**Charging IC:** Must be I²C controllable for programmable charge current (to adapt to different battery sizes).

---

## 4. Sensing Interfaces

### 4.1 mmWave Radar Interface
**Protocol:** SPI (Mode 0/3) or UART.
**Speed:** Unconstrained. Supports SPI > 20 MHz or UART > 4 Mbps.
**Control GPIOs:**
*   `RADAR_nRST` (Output) — Hardware reset.
*   `RADAR_nCS` (Output) — Chip select (SPI).
*   `RADAR_IRQ` (Input) — Data ready interrupt.

**Power:** Dedicated low-noise LDO recommended.

---

### 4.2 IMU (Inertial Measurement Unit) Interface
**Protocol:** I²C (up to 1 MHz Fast Mode Plus) or SPI (up to 10 MHz).
**Data Ready:** `IMU_DRDY` (Input) — Interrupt to wake MCU on new sample.

**Power:** Ultra-low-power rail (`VCC_1V8`) preferred.

---

### 4.3 Acoustic Interface (Microphone Array)
**Protocol:** I²S (Inter-IC Sound) or PDM (Pulse Density Modulation).
**Channels:** 2–4 microphones per node.
**Clocking:** MCU provides Master Clock (MCLK) and Bit Clock (BCLK).

---

### 4.4 LiDAR / ToF Interface
**Protocol:** UART (common for integrated modules) or SPI.
**Speed:** Configurable baud rate (e.g., 115200, 921600, up to module max).
**Trigger:** `LIDAR_TRIG` (Output) — External trigger input for synchronization.

---

### 4.5 Environmental Sensors
**Protocol:** I²C.
**Sensors:** Temperature, Humidity, Pressure, Gas (VOC).
**Integration:** Shared I²C bus with IMU (if address conflicts are avoided).

---

### 4.6 Event Sensor Interface
**Protocol:** MIPI CSI-2 (for high bandwidth) or proprietary SPI-like parallel.
**Data Rate:** Very high. Requires DMA support.
**Lens Mount:** M12 or integrated lens.

---

## 5. Identification Module Interface (Pendant Node Only)

This interface is distinct from the sensing mesh. It is for high-fidelity optical/biometric capture.

### 5.1 Camera Sensor Interface
**Protocol:** MIPI CSI-2 (2-lane or 4-lane) or parallel interface.
**Resolution:** **Unconstrained.** Supports any sensor compatible with the physical interface (VGA to 4K+).

### 5.2 Identification Processor Interface (Optional)
If off-loading processing:
**Protocol:** USB 2.0/3.0 or PCIe (for advanced co-processors).

### 5.3 Hardware Kill Switch (CRITICAL)
**Implementation:**
1.  **Physical Switch:** A mechanical slider or button accessible to the user.
2.  **Power Path Control:** The switch physically disconnects `VCC_3V3` and `VCC_1V8` to the Identification Module. It must **NOT** be a software GPIO read.
3.  **Firmware Check:** MCU reads a separate GPIO (`ID_KILL_STATUS`) to detect switch state at boot. If switch is "Disabled," MCU MUST NOT attempt to communicate with the Identification Module.

```
  Battery ──[ Switch ]──> Identification Module Power Rail
                               │
                               └─> ID_KILL_STATUS (GPIO Input to MCU, pulled low when power cut)
```

---

## 6. Actuator Interface (Haptic)

**Driver:** H-Bridge or specialized haptic driver IC.
**Interface:** PWM from MCU or I²C controller.
**Amplitude Control:** Via PWM duty cycle or driver I²C registers.
**Feedback:** `HAPTIC_SENSE` (Input) — Back-EMF for active sensing (optional).

---

## 7. Body-Area Network (BAN) Radio

### 7.1 Radio Module Interface
**Protocol:** UART, SPI, or SDIO.
**Antenna:** PCB trace antenna on main board or U.FL connector for external antenna.

### 7.2 Protocol Stack
**Implementation:** Firmware.
**Standards:**
*   BLE 5.x (Long Range, Coded PHY).
*   UWB (Ultra-Wideband) for high-precision time-sync / ranging.
*   Proprietary ISM band protocols.

---

## 8. Node-Specific Configurations

### 8.1 Pendant Node
**Sensors:** mmWave Radar, Acoustic (2-4 mics), IMU, **Identification Module**.
**Actuators:** Haptic.
**Power:** Highest capacity (e.g., 500-1000 mAh).
**Connectors:** USB-C for charging/debug.

### 8.2 Bracelet Node
**Sensors:** mmWave Radar, IMU.
**Actuators:** Haptic.
**Power:** 150-300 mAh.

### 8.3 Belt Node
**Role:** Primary Compute & Fusion Hub.
**Sensors:** mmWave Radar, IMU, Environmental.
**Compute:** High-performance MCU or SoC (Linux-capable optional).
**Connectivity:** Wi-Fi (optional), BLE + UWB.
**Power:** Highest capacity (2000-5000 mAh).
**Connectors:** USB-C, Expansion Header for additional nodes.

### 8.4 Anklet Node
**Sensors:** ToF/LiDAR, IMU.
**Actuators:** Haptic.
**Power:** 150-300 mAh.

### 8.5 Eyewear Node
**Sensors:** Event Sensor, IMU.
**Power:** Integrated into frame arms.

---

## 9. Debug & Development Interface

**Connector:** 10-pin ARM Cortex Debug Connector (0.05" pitch) or Tag-Connect footprint.
**Signals:**
*   `SWDIO`, `SWCLK`
*   `nRST`
*   `SWO` (Serial Wire Output for printf)
*   `UART_TX`, `UART_RX` (Console)
*   `VCC`, `GND`

**Bootloader:** USB DFU or ROM bootloader for firmware updates without debug probe.

---

## 10. Storage Interface

**Primary:** Internal MCU Flash.
**Extended:** microSD Card slot (SPI mode) or External QSPI Flash.
**Capacity:** **Unconstrained.** User configures based on logging needs.

---

## 11. Mechanical Integration

**Mounting:**
*   **Bracelet:** Standard watch strap pins (20mm/22mm lug width).
*   **Pendant:** Necklace loop integrated into enclosure.
*   **Anklet:** Velcro or snap closure.
*   **Belt:** Belt clip or integrated into belt buckle.

**Environmental:**
*   **Sealing:** IP67 recommended (water/dust resistant).
*   **Skin Contact:** Hypoallergenic materials (Titanium, Surgical Steel, Hypoallergenic Polymers).
