# Ultra-Low-Power Edge Impulse (EI) Data Logger for Cold-Start Monitoring

## 💡 Overview

This project combines ultra-low-power system design, embedded sensing, and TinyML-ready data logging. It focuses on measuring and analysing the **Time-to-First-Measurement (TTFM)** of an ESP32-based node waking from deep sleep — a key performance parameter in energy-constrained IoT systems.

Using the **XIAO ESP32-S3** (or a similar ESP32 variant), we build a standalone, battery-powered logger that periodically wakes, captures sensor readings (Lux, Temperature, Pressure), measures the time required to produce that reading after wakeup, and stores the result in a dataset suitable for Edge Impulse (EI) or other ML pipelines.

The goal is to determine if environmental factors (like temperature) have a measurable, non-linear impact on the chip's cold-start performance.

## 🎯 Goal

Create an **ultra-low-power data logging node** that:

* Wakes from deep sleep via **RTC timer** (currently set to 60-second intervals).

* Measures **Time-to-First-Measurement (TTFM)** with high accuracy (microsecond resolution).

* Reads environmental sensor data (Lux, Temperature, Pressure).

* Logs data in **ML-ready CSV format** for Edge Impulse or Python analysis.

* Operates on a small **LiPo battery** with minimal power consumption.

## 🛠️ Technical Details & Wiring Connections

This setup uses two primary I2C sensors (BH1750 and BMP180) wired to a single I2C bus on the XIAO ESP32-S3. The specific I2C pins used in the accompanying code (`sensors_read_debug.ino`) are defined below.

| Component | Function | I2C Address | XIAO ESP32-S3 Pin | Code Definition |
| :--- | :--- | :--- | :--- | :--- |
| **XIAO ESP32-S3** | Microcontroller/RTC | N/A | N/A | N/A |
| **BH1750 (Lux)** | Light Sensor | `0x23` | D43 (SDA) / D44 (SCL) | `#define SDA_PIN 43` |
| **BMP180** | Temp/Pressure Sensor | `0x77` | D43 (SDA) / D44 (SCL) | `#define SCL_PIN 44` |
| **Storage** | MicroSD Card | SPI (Optional) | N/A | N/A |
| **Power** | 3.7V LiPo Battery | N/A | JST Connector | N/A |

### Wiring Diagram

Connect the VCC and GND of both the BH1750 and BMP180 to the 3.3V and GND pins of the XIAO. The I2C data lines should be connected as follows:

| Sensor Pin | XIAO ESP32-S3 Pin |
| :--- | :--- |
| **SDA** (Data) | **D43** (GPIO 43) |
| **SCL** (Clock) | **D44** (GPIO 44) |

---

## ⚙️ Implementation Steps

### 1. Ultra-Low-Power Wakeup Module (C++ / Arduino)

* The core of the firmware (`ULP_EI_TTFM_Logger.ino`) implements a **deep-sleep → wake → measure → log → sleep** state machine.

* The wake source is the **RTC timer** (`esp_sleep_enable_timer_wakeup`).

* **TTFM Measurement:** The time is measured using `esp_timer_get_time()` to achieve microsecond resolution, specifically capturing the time from the sensor command to the successful data read.

### 2. Data Acquisition & TinyML-Ready Logging (C++ / Edge Impulse)

On each wake cycle:

1. **Start TTFM Timer:** Begin timing immediately before sending the BH1750 (Lux) measurement command.

2. **Capture Sensor Data:** Read Lux (BH1750) after its $180\text{ms}$ integration time, then read Temperature/Pressure (BMP180).

3. **Stop TTFM Timer:** Record the total time once the Lux value is successfully acquired.

4. **Log to CSV:** Store the data on the microSD card in the following CSV format:
   
Timestamp_ms, Lux_Value, Temperature_C, Pressure_Pa, TTFM_us

5. **Re-enter Deep Sleep:** Program the next wake time and call `esp_deep_sleep_start()`.
### 3. Safety/Boot Feature (ESP32-S3 Specific)

Due to the sensitive nature of the ESP32-S3's native USB port during deep sleep, a **SAFE BOOT** feature is implemented:

* **Behavior when connected to PC via USB-C:** After a fresh flash, the device remains awake, waiting for the character **`C`** via the Serial Monitor. This prevents the logger from entering Deep Sleep immediately and becoming unreachable for debugging/re-flashing.
* **Behavior when powered by Battery/Power Pack:** If no USB-C connection is active, the device **automatically bypasses** the wait and initiates the first Deep Sleep cycle immediately after `setup()` completes.

---

## 📉 Expected Performance Degradation (Hypothesis)

Initial log data (from $16^\circ\text{C}$ to $30^\circ\text{C}$) shows the TTFM is extremely stable, centered around the $\mathbf{180,000 \text{ us}}$ BH1750 sensor delay. This stability is expected to be non-linear across a wider thermal range.

### Stability Rationale

The ESP32-S3's internal RTC (Real-Time Clock) oscillator includes **Temperature Compensation Circuits (TCCs)**. These circuits actively adjust the oscillator frequency to maintain time integrity.

* **Observed Behavior:** The stability of the $\approx 181 \text{ ms}$ TTFM across the tested range confirms the TCCs effectively counteract thermal drift in typical ambient conditions.

### Expected Non-Linear Degradation

The performance is hypothesized to degrade sharply only when the device is stressed beyond the TCC's effective range, typically approaching freezing temperatures. 

| Condition | Expected TTFM Trend | Primary Cause of Degradation |
| :--- | :--- | :--- |
| **$40^\circ\text{C}$ to $10^\circ\text{C}$** | Stable (Flat Baseline) | Fixed sensor delay dominates; Chip compensation is fully effective. |
| **$0^\circ\text{C}$ and Below** | **Sharp Increase** (Non-linear) | 1. **Slower Oscillator Startup:** Extreme cold degrades silicon physics, increasing the time required for the internal clock to achieve stable oscillation. 2. **Battery Voltage Sag:** LiPo battery internal resistance rises non-linearly when cold, causing severe voltage drops during the instant power spike of wake-up. This extends the boot time. |

The next step is to test at $\mathbf{\approx 5^\circ\text{C}}$ (refrigerator) to identify the low-temperature point where the TTFM stability begins to degrade.
