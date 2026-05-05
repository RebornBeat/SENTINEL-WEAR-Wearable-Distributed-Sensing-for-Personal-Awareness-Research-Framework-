# Pendant Node — Hardware Reference Design

**Project:** SENTINEL-WEAR
**Node Type:** Primary Sensing & Identification Node (Upper Hemisphere Coverage)
**Form Factor:** Pendant / Chest-Mounted
**Status:** Reference Design v1.0
**License:** CERN-OHL-S v2

---

## 1. Overview

The Pendant Node is the primary upper-hemisphere sensing unit in the SENTINEL-WEAR architecture. Positioned on the chest, it provides:
- **360° Azimuth Coverage:** Forward-facing hemisphere.
- **Elevation Coverage:** Upward and forward-looking.
- **Primary Identification Layer:** Hosts the optional Identification Module.
- **Primary Compute:** In some configurations, can act as a secondary compute node.

This directory contains the KiCad schematic and PCB layout files for the Pendant Node. It is a **reference design**. Researchers are free to substitute components, increase sensor capabilities, or modify the form factor to suit their specific experimental needs.

---

## 2. Design Philosophy

### 2.1 Limitless Configuration
This design does not impose artificial constraints on capabilities.
- **Sensors:** The reference design uses a specified set of sensors, but the PCB layout accommodates alternative footprints where possible.
- **Processing:** The MCU selection provides ample headroom for algorithm development.
- **Connectivity:** No hardcoded connectivity assumptions. The node supports local-only logging, BLE streaming, and Wi-Fi backhaul (via a module).

### 2.2 Sensing vs. Identification
The Pendant Node physically separates the **Sensing Layer** from the **Identification Layer**:
- **Sensing Layer:** Always-on, privacy-preserving (mmWave, IMU, Acoustic). Powered by the main power rail.
- **Identification Layer:** The camera/biometric module. **Controlled by a dedicated hardware kill switch (SW1)**. The MCU firmware must read the kill-switch state at boot and halt execution if the switch is in the "disabled" position.

---

## 3. High-Level Block Diagram

```
[ Power Input / Charger ]
          │
    [ PMIC / Power Mgmt ]───────────┐
          │                         │
          ▼                         ▼
[ Main MCU (nRF5340 / STM32H7) ]   [ haptic driver ]
          │
          ├───[ mmWave Radar (I2C/SPI) ]
          │
          ├───[ IMU (SPI) ]
          │
          ├───[ Microphone Array (I2S/PDM) ]
          │
          ├───[ Environmental Sensors (I2C) ]
          │
          ├───[ Event Sensor (SPI) ] (Optional)
          │
          └───[ Identification Module ]
                 │
                 ├───[ Camera Module (MIPI CSI / SPI) ]
                 │
                 └───[ Kill Switch (GPIO) ]
```

---

## 4. Component Selection (Reference Design v1.0)

### 4.1 Microcontroller (MCU)
**Reference Part:** Nordic Semiconductor nRF5340
- **Architecture:** Dual-core Arm Cortex-M33 (Application Core + Network Core)
- **Clock:** 128 MHz (App Core), 64 MHz (Net Core)
- **Flash:** 1 MB Application Flash, 256 KB RAM
- **Connectivity:** Bluetooth 5.1 / LE Audio, IEEE 802.15.4 (Thread), SPI, I2C, I2S, PDM, UART.
- **Reasoning:** Integrated BLE radio (critical for BAN), high-performance core for sensor fusion, ample memory for logging and algorithm development.

### 4.2 Sensors

#### 4.2.1 mmWave Radar
**Reference Part:** Texas Instruments IWR6843ISK-ODS or Infineon BGT60LTR11AIP
- **Interface:** SPI + GPIOs (Data Ready, Reset).
- **Frequency:** 60–64 GHz.
- **Features:** On-chip antenna, presence detection, gesture recognition.
- **Power:** 3.3V.

#### 4.2.2 Inertial Measurement Unit (IMU)
**Reference Part:** TDK InvenSense ICM-42688-P
- **Interface:** SPI.
- **Gyro Range:** ±2000 dps.
- **Accel Range:** ±16 g.
- **Features:** Low noise, high stability, on-chip FIFO.
- **Purpose:** Node orientation, gesture detection, pedestrian dead reckoning.

#### 4.2.3 Acoustic Array
**Reference Part:** Knowles SPH0645LM4H-B (x2) or equivalent PDM microphones.
- **Interface:** PDM (connected to MCU I2S/PDM peripheral).
- **Configuration:** 2-microphone array (spaced ~20mm apart) for basic direction-of-arrival (DOA) estimation.

#### 4.2.4 Environmental Sensors (Optional)
**Reference Part:** Bosch BME680
- **Interface:** I2C or SPI.
- **Measures:** Temperature, Humidity, Pressure, Gas (VOC).
- **Purpose:** Contextual awareness.

#### 4.2.5 Event Sensor (Optional / Advanced)
**Reference Part:** Sony IMX637 (Prophesee EVK) or iniVation DAVIS346.
- **Interface:** MIPI CSI-2 or USB (depending on module).
- **Purpose:** High-speed transient detection (extreme velocity research track).
- **Note:** This is an advanced research option. The base design may route high-speed differential pairs to a connector for an external module.

### 4.3 Identification Module (Opt-In Layer)
**Reference Part:** Raspberry Pi Camera Module 3 or Arducam IMX477.
- **Interface:** MIPI CSI-2 (direct to MCU) or SPI (for lower-res modules).
- **Control:**
    - **Power:** Enabled/disabled by a load switch controlled by the MCU.
    - **Kill Switch (SW1):** A physical slider switch that disconnects the Identification Module's power rail and data lines directly. **The MCU must sample this switch at boot.** If the switch is OFF, the MCU must not attempt to initialize the camera driver.

### 4.4 Power Management
**Reference Part:** Nordic nPM1300 PMIC
- **Features:** Battery charging (LiPo/Li-Ion), Buck regulators, Fuel Gauge.
- **Battery:** Single-cell LiPo (300–600 mAh typical for pendant form factor).
- **Charging:** USB-C (5V).

### 4.5 Haptics
**Reference Part:** TI DRV2605L Haptic Driver.
- **Interface:** I2C.
- **Actuator:** LRA (Linear Resonant Actuator).

---

## 5. Pin Mapping & Configuration

*(Note: This is a reference mapping for the nRF5340. Adjust based on your specific MCU selection and pin availability.)*

### 5.1 mmWave Radar (IWR6843)
| Signal | nRF5340 Pin | Description |
| :--- | :--- | :--- |
| `SPI_MOSI` | P1.00 | SPI Master Out Slave In |
| `SPI_MISO` | P1.01 | SPI Master In Slave Out |
| `SPI_CLK` | P1.02 | SPI Clock |
| `RADAR_CS` | P1.03 | Chip Select |
| `RADAR_RST` | P1.04 | Reset |
| `RADAR_IRQ` | P1.05 | Data Ready Interrupt |

### 5.2 IMU (ICM-42688-P)
| Signal | nRF5340 Pin | Description |
| :--- | :--- | :--- |
| `IMU_CS` | P1.06 | Chip Select |
| `IMU_INT` | P1.07 | Data Ready Interrupt |
| `SPI_MOSI` | P1.00 | (Shared with Radar) |
| `SPI_MISO` | P1.01 | (Shared with Radar) |
| `SPI_CLK` | P1.02 | (Shared with Radar) |

### 5.3 Acoustic (PDM Microphones)
| Signal | nRF5340 Pin | Description |
| :--- | :--- | :--- |
| `PDM_CLK` | P1.08 | Clock Output |
| `PDM_DATA` | P1.09 | Data Input |

### 5.4 Identification Module (Camera)
| Signal | nRF5340 Pin | Description |
| :--- | :--- | :--- |
| `CAM_PWDN` | P1.10 | Power Down Control |
| `KILL_SW` | P1.11 | **Hardware Kill Switch Input** (Read at boot) |
| `MIPI_CSI_CLK+` | Dedicated | High-speed diff pair |
| `MIPI_CSI_D0+` | Dedicated | High-speed diff pair |

### 5.5 Power, Haptics, & Debug
| Signal | nRF5340 Pin | Description |
| :--- | :--- | :--- |
| `HAPTIC_PWM` | P1.12 | PWM to DRV2605L |
| `I2C_SDA` | P1.13 | I2C Data |
| `I2C_SCL` | P1.14 | I2C Clock |
| `SWD_CLK` | P0.18 | Debug |
| `SWD_DIO` | P0.20 | Debug |
| `USB_D+` | Dedicated | USB Data+ |
| `USB_D-` | Dedicated | USB Data- |
| `BAT_SENSE` | AIN0 | Battery Voltage (via divider) |

---

## 6. Power Architecture

### 6.1 Power Rails
- **V_BAT:** Direct battery connection (3.7V - 4.2V).
- **V_SYS:** System power (3.3V), regulated by nPM1300.
- **V_RADAR:** 3.3V, always-on.
- **V_IMU:** 3.3V, always-on.
- **V_MCU:** 3.3V, always-on.
- **V_CAM:** 3.3V or 2.8V (camera dependent), **switched by MOSFET controlled by MCU + Kill Switch.**

### 6.2 Kill Switch Logic (CRITICAL)
The Kill Switch (`SW1`) is a physical DPST (Double Pole Single Throw) slider switch.
1.  **Pole 1:** Breaks the `V_CAM` power rail.
2.  **Pole 2:** Breaks the MIPI data lines or connects a `KILL_SW` signal to MCU GPIO.

**Firmware Requirement:**
- The MCU **must** read the `KILL_SW` GPIO pin early in the boot process (before initializing any peripherals).
- If `KILL_SW` is LOW (switch OFF), the MCU **must halt** or enter a safe state and **never attempt to power on or communicate with the Identification Module.**

---

## 7. Form Factor Constraints

- **Dimensions:** Maximum 50mm x 50mm (excluding mounting lugs/loop).
- **Height:** < 15mm populated height.
- **Weight Target:** < 70g (including battery).
- **Enclosure:** Designed to fit within a jewelry-grade pendant enclosure. The CAD files for the mechanical enclosure are in `/mechanical/cad/pendant_enclosure.step`.

---

## 8. Bill of Materials (BOM) Overview

A full BOM is available in `hardware/bom/pendant_node_bom.csv`.

**Key Components Summary:**
- U1: nRF5340-QKAA-AB0R (MCU)
- U2: IWR6843ISK-ODS (Radar)
- U3: ICM-42688-P (IMU)
- U4: nPM1300 (PMIC)
- U5: DRV2605L (Haptic Driver)
- MK1, MK2: SPH0645LM4H-B (Microphones)
- J1: USB-C Connector
- SW1: DPST Slide Switch (Kill Switch)

---

## 9. Layout Guidelines

1.  **RF Section:** The mmWave radar antenna area must be free of copper and components on all layers. Follow the antenna keep-out recommendations from the IWR6843 datasheet strictly.
2.  **High-Speed Signals:** Route MIPI CSI traces with controlled impedance (typically 100Ω differential). Length-match the pairs.
3.  **Power:** Use adequate decoupling capacitors close to all power pins of the MCU and Radar.
4.  **Grounding:** Use a solid ground plane. Star grounding is recommended to prevent digital noise from coupling into the sensitive analog/mic inputs.

---

## 10. Hardware Configuration Options

The following resistors/jumpers configure the board:

- **J2:** Radar SPI Chip Select source (MCU or external debugger).
- **R43:** Pull-up resistor for I2C bus (Populate if needed).
- **R44:** Camera Power Down pull-down (Default OFF).
