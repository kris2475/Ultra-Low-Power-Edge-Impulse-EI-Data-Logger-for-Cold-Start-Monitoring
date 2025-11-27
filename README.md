# Ultra-Low-Power Edge Impulse (EI) Data Logger for Cold-Start Monitoring  
*A low-budget, pre-Christmas embedded ML project*

## 💡 Overview  
This project combines ultra-low-power system design, embedded sensing, and TinyML-ready data logging. It focuses on measuring and analysing the **Time-to-First-Measurement (TTFM)** of an ESP32-based node waking from deep sleep — a key performance parameter in energy-constrained IoT systems.

Using an ESP32/XIAO ESP32S3 Sense (or any ESP32 variant), you will build a standalone, battery-powered logger that periodically wakes, captures a sensor reading, measures the time required to produce that reading after wakeup, and stores the result in a dataset suitable for Edge Impulse (EI) or other ML pipelines.

The system is intentionally minimal-hardware and low-cost, using mostly components you likely already have.

---

## 🎯 Goal  
Create an **ultra-low-power data logging node** that:

- Wakes from deep sleep via **RTC timer** or **external trigger**  
- Measures **Time-to-First-Measurement (TTFM)** with high accuracy  
- Reads environmental sensor data (e.g., temperature, humidity, light, or audio)  
- Logs data in **ML-ready CSV/JSON format** for Edge Impulse or Python analysis  
- Operates on a small **LiPo battery** with minimal power consumption  

---

## 🛠️ Technical Details & Budget

| Area             | Detail                                                                                  | Cost Estimate                  |
|------------------|------------------------------------------------------------------------------------------|--------------------------------|
| **Microcontroller** | XIAO ESP32S3 Sense (recommended) or standard ESP32/ESP32-S3 dev board                    | ≈ £10–£15 (likely already owned) |
| **Sensor**          | Built-in sensors on dev board, or external BME280/BMP180                               | ≈ £0–£5                        |
| **Storage**         | Internal Flash/PSRAM, or optional microSD                                               | ≈ £0–£5                        |
| **Power**           | Single LiPo cell                                                                        | ≈ £5                           |
| **Total**           | **Potentially < £20** using existing hardware                                           |                                |

> **“Export to Sheets”** — copy/paste the table directly into Google Sheets or Excel.

---

## ⚙️ Implementation Steps  

### 1. Ultra-Low-Power Wakeup Module (C++ / Arduino)
- Implement a **deep-sleep → wake → measure → log → sleep** state machine.  
- Use **RTC timer** or external GPIO interrupt as the wake source.  
- Measure TTFM using:  
  - `rtc_time_get()` or low-level RTC registers  
  - fallback: `esp_timer_get_time()` for microsecond resolution  
- Goal: quantify the time between **wake event** and **successful sensor read**.

**Key focus**: wake time variance under different sleep intervals, temperatures, and wake sources.

---

### 2. Data Acquisition & TinyML-Ready Logging (C++ / Edge Impulse)  
On each wake cycle:

1. Capture selected sensor data:  
   - temperature / humidity (BME280)  
   - light (ADC input)  
   - microphone PDM sample (Sense board)  
2. Log:  
   ```text
   timestamp, TTFM_ms, sensor_reading
