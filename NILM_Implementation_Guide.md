# NILM-Based Smart Energy Monitor — Complete Implementation & Architecture Guide

> **Project Title:** Power Line Fingerprinting — Non-Intrusive Load Monitoring (NILM) Smart Energy Monitor  
> **Target Audience:** Engineering Team, Project Supervisors, & Implementation Developers  
> **Document Purpose:** Primary Technical & Architectural Blueprint for Prototype 1 & Beyond  

---

## Executive Summary & System Overview

Non-Intrusive Load Monitoring (NILM) aims to disaggregate total electrical consumption into individual appliance-level energy signatures using a minimal number of central sensing points. 

This document defines the frozen system architecture, hardware/software stack, data contracts, signal processing workflow, machine learning pipeline, database schemas, and frontend/backend integration strategy for our major project.

---

## 1. Recommended Overall Architecture

The architecture follows a decoupled, event-driven microservices model. High-speed raw electrical telemetry is acquired at the hardware edge, transmitted via a lightweight messaging bus, processed centrally in Python using advanced signal decomposition and machine learning, orchestrated via Node-RED, stored in a time-series database, and exposed to end users via a real-time web interface.

### Recommended System Flow

```
+-------------------------------------------------------------------------------+
|                             PHYSICAL LAYER                                    |
|  [ Mains AC Electrical Line ] ---> [ SCT-013 CT Clamp & ZMPT101B Voltage ]    |
+-------------------------------------------------------------------------------+
                                       | Analog Signals (Voltage & Current)
                                       v
+-------------------------------------------------------------------------------+
|                            EDGE ACQUISITION LAYER                             |
|  [ ESP32 Microcontroller ] (Synchronous 8-10 kHz Sampling & Buffer Windowing) |
+-------------------------------------------------------------------------------+
                                       | Wi-Fi / MQTT Waveform Packets
                                       v
+-------------------------------------------------------------------------------+
|                             COMMUNICATION LAYER                               |
|  [ Mosquitto MQTT Broker ] (Topic Routing & Message Decoupling)               |
+-------------------------------------------------------------------------------+
         | nilm/device01/waveform                      ^ nilm/device01/result
         v                                             |
+-------------------------------------------------------------------------------+
|                       SIGNAL PROCESSING & ML SERVICE                          |
|  [ Python Service ]                                                           |
|    |--> Signal Normalization (Z-score / Min-Max)                              |
|    |--> Variational Mode Decomposition (VMD -> IMFs)                           |
|    |--> Feature Extraction (Harmonics, P, Q, PF, THD, Transient Features)     |
|    |--> TFLite ML Engine (Appliance Classification & Power Estimation)        |
+-------------------------------------------------------------------------------+
                                       |
                                       v (nilm/device01/result)
+-------------------------------------------------------------------------------+
|                         ORCHESTRATION & STORAGE LAYER                         |
|  [ Node-RED ] ---> (Rule Evaluation, Alert Logic & InfluxDB Data Formatting)   |
|         |                                                                     |
|         v                                                                     |
|  [ InfluxDB ] (Time-Series Storage: kWh, Power, States, Confidence Scores)    |
+-------------------------------------------------------------------------------+
                                       | Flux / REST Queries
                                       v
+-------------------------------------------------------------------------------+
|                             BACKEND API GATEWAY                               |
|  [ FastAPI Backend ] <---> [ WebSocket Server ]                               |
+-------------------------------------------------------------------------------+
                                       | HTTP REST & WebSockets
                                       v
+-------------------------------------------------------------------------------+
|                           USER INTERFACE LAYER                                |
|  [ React + Vite Dashboard ] (Recharts Visualization & Real-Time Alerts)       |
+-------------------------------------------------------------------------------+
```

---

## 2. Technology Stack

| Component / Layer | Technology | Purpose & Responsibility |
| :--- | :--- | :--- |
| **Edge Hardware** | ESP32 (32-bit Dual Core) | High-speed synchronous sampling of AC voltage and current waveforms. |
| **Current Sensor** | SCT-013-000 CT Clamp | Non-invasive current measurement via electromagnetic induction. |
| **Voltage Sensor** | ZMPT101B Module | Precision voltage transformer module with op-amp signal conditioning. |
| **Firmware Dev** | Arduino IDE / PlatformIO | Writing C++ firmware, configuring hardware timers, ADC, and Wi-Fi/MQTT. |
| **Network Protocol** | Wi-Fi (802.11 b/g/n) | Wireless local network communication layer between ESP32 and broker. |
| **Messaging Protocol** | MQTT (v3.1.1/v5.0) | Low-overhead, publish-subscribe telemetry transport protocol. |
| **MQTT Broker** | Eclipse Mosquitto | Central message broker handling pub/sub routing between edge, ML, & Node-RED. |
| **ML/DSP Language** | Python 3.10+ | Scientific processing runtime for signal decomposition, feature extraction, and ML. |
| **Signal Processing** | VMD (`vmdpy` / Scipy) | Variational Mode Decomposition to extract intrinsic mode functions (IMFs). |
| **ML Inference** | TensorFlow Lite (TFLite) | Lightweight execution engine for quantized neural networks / classifiers. |
| **Orchestrator** | Node-RED | Visual flow-based IoT middleware for alert logic, routing, and DB writing. |
| **Database** | InfluxDB (v2.x) | High-performance time-series database optimized for energy metrics. |
| **Backend Framework** | FastAPI (Python) | High-throughput async REST API and WebSocket gateway for dashboard data. |
| **Frontend UI** | React + Vite | Modern, responsive single-page web app for real-time energy monitoring. |
| **Data Viz** | Recharts / ECharts | Interactive real-time and historical charting library for power and cost metrics. |
| **IDE / Tooling** | VS Code | Integrated development environment for Python, Node.js, and web code. |
| **Containerization** | Docker Compose | Local multi-container orchestration for Mosquitto, Node-RED, InfluxDB, & FastAPI. |

---

## 3. Physical Data Acquisition

Physical data acquisition captures AC current and AC voltage waveforms safely and accurately.

```
+------------------+         +-------------------+
| Appliance Load   |         | Mains AC Voltage  |
| Current (Amps)   |         | 230V RMS (50Hz)   |
+--------+---------+         +---------+---------+
         |                             |
         v                             v
+------------------+         +-------------------+
| SCT-013 CT Clamp |         | ZMPT101B Voltage  |
| Current Sensor   |         | Sensor Module     |
+--------+---------+         +---------+---------+
         |                             |
         | Analog Voltage (0-3.3V)     | Analog Voltage (0-3.3V)
         | Biased at 1.65V DC          | Biased at 1.65V DC
         +--------------+--------------+
                        |
                        v
         +-----------------------------+
         | ESP32 Analog Inputs (ADC)   |
         | GPIO 34 (I) & GPIO 35 (V)   |
         +-----------------------------+
```

### Sensor Roles Explained

1. **SCT-013-000 CT Clamp**: A non-invasive split-core Current Transformer. When clamped around an AC line wire (Phase/Live only), the primary AC current induces a proportional secondary current. Passing this through a burden resistor converts it into an AC voltage signal proportional to the load current.
2. **ZMPT101B Voltage Sensor Module**: A micro voltage transformer paired with an operational amplifier circuit. It steps down 230V RMS mains AC to a safe low-voltage AC signal and applies a DC offset (1.65V) so that the entire sine wave fits safely within the 0V–3.3V input range of the ESP32 ADC.

> [!CAUTION]
> **ELECTRICAL SAFETY WARNING: HIGH VOLTAGE HAZARD**  
> Mains AC voltage (110V–240V RMS) is **lethal**. Strictly follow safety protocols:
> * **Never** expose bare live wires or touch energized terminals.
> * Always clamp the SCT-013 CT around a **single conductor** (Live/Phase line only), never around a double-insulated jacket containing both Live and Neutral.
> * Ensure the ZMPT101B module is enclosed in an insulated protective case.
> * Perform all circuit modifications with the main circuit breaker turned **OFF**.

---

## 4. ESP32 Data Acquisition & Edge Behavior

In the initial implementation, the ESP32 serves strictly as a high-speed **Data Acquisition Unit (DAQ)**. Advanced math (VMD & ML) is deliberately offloaded to Python.

```
         ESP32 DAQ LOOP (Sampling @ 8-10 kHz)
  +-------------------------------------------------+
  | 1. Read ADC Channel 1 (Voltage on GPIO 35)      |
  | 2. Read ADC Channel 2 (Current on GPIO 34)      |
  | 3. Store (V, I) pair into Ping-Pong Buffer      |
  | 4. Microsecond Timer Delay (dt ≈ 100-125 µs)   |
  +------------------------+------------------------+
                           |
                           v Buffer Full (1024 Samples)
  +-------------------------------------------------+
  | 5. Timestamp Waveform Window (NTP / micros)     |
  | 6. Package Data Packet                          |
  | 7. Publish to MQTT (`nilm/device01/waveform`)   |
  +-------------------------------------------------+
```

### ESP32 Responsibilities

* **Synchronous Sampling**: Samples voltage and current channels simultaneously at target rate of **8–10 kHz** (approx. 160–200 samples per 50 Hz AC cycle).
* **Waveform Buffering**: Collects contiguous waveform windows (e.g., 512 or 1024 sample pairs, representing ~5–10 complete AC cycles).
* **Timestamping**: Attaches precise millisecond/microsecond timestamps to each window.
* **Transmission**: Transmits packaged windows over Wi-Fi via MQTT to the broker.
* **Device Heartbeat**: Periodically publishes status telemetry (`nilm/device01/status`) containing Wi-Fi RSSI, uptime, and free heap memory.

---

## 5. MQTT Communication Architecture

MQTT is a lightweight, asynchronous publish-subscribe message transport protocol ideal for high-frequency IoT sensor streams.

```
 [ ESP32 (DAQ) ] ----Publishes Waveforms----> [ MQTT Broker ] ----Subscribes----> [ Python ML Service ]
                                              [ (Mosquitto) ] <---Publishes Results----+
                                                     |
                                                     +----Subscribes Results----> [ Node-RED ]
```

### Proposed MQTT Topic Tree

| Topic | Publisher | Subscriber(s) | Description |
| :--- | :--- | :--- | :--- |
| `nilm/device01/waveform` | ESP32 | Python ML Service | High-speed raw voltage & current buffer windows. |
| `nilm/device01/status` | ESP32 | Node-RED, FastAPI | Device health, IP address, uptime, heap, RSSI. |
| `nilm/device01/result` | Python ML Service | Node-RED, Dashboard | Disaggregation results: appliance IDs, power, confidence. |
| `nilm/device01/alert` | Node-RED | FastAPI, WebSockets | System warnings (overload, appliance left ON). |

### Example Payloads

#### 1. Raw Waveform Payload (`nilm/device01/waveform`)
```json
{
  "device_id": "esp32_sensor_01",
  "timestamp": 1776595200123,
  "sampling_rate_hz": 8192,
  "window_size": 512,
  "v_samples": [1.65, 1.82, 1.98, 2.11, 2.20, 2.24, 2.20, 2.11, 1.98],
  "i_samples": [1.65, 1.68, 1.74, 1.82, 1.89, 1.91, 1.89, 1.82, 1.74]
}
```

#### 2. Classification Result Payload (`nilm/device01/result`)
```json
{
  "device_id": "esp32_sensor_01",
  "timestamp": 1776595200850,
  "total_real_power_w": 1845.2,
  "total_reactive_power_var": 312.4,
  "appliances": [
    {
      "name": "Geyser",
      "state": "ON",
      "power_w": 1800.0,
      "confidence": 0.96
    },
    {
      "name": "Fan",
      "state": "ON",
      "power_w": 45.2,
      "confidence": 0.89
    }
  ]
}
```

---

## 6. Optimization Strategy: Waveform Windowing vs. Per-Sample Messaging

Sending individual MQTT packets per ADC sample at 8–10 kHz is inefficient and will crash the network stack.

### Why Per-Sample Messaging Fails

* At 10 kHz, sending 10,000 MQTT messages per second generates severe overhead.
* TCP/IP + MQTT packet headers (~40–60 bytes) exceed the sample payload size (2–4 bytes), wasting over **90% of network bandwidth**.
* High packet rates saturate the ESP32 TCP stack, causing buffer overflows, dropped packets, and Wi-Fi disconnects.

### Windowed Buffer Solution

```
Individual Samples (10,000 / sec)
  (V1,I1)  (V2,I2)  (V3,I3) ... (V1024,I1024)
     |        |        |            |
     +--------+--------+------------+
                      |
                      v Accumulated into Ring Buffer
           [ Waveform Window Buffer ] (1024 Samples = ~100 ms of data)
                      |
                      v Packaged into 1 Payload
           [ Single MQTT Packet Sent Every 100-500 ms ]
```

### Payload Format Evolution

1. **Development Phase**: Use JSON arrays for human-readable debugging via tools like MQTT Explorer.
2. **Production Optimization**: Migrate to **binary payload structures** (e.g., packed `int16_t` arrays or Protocol Buffers). Binary encoding reduces payload size by **~75%** and lowers ESP32 serialization overhead.

---

## 7. Python Signal Processing & ML Service Architecture

The Python ML service consumes raw waveforms, executes Variational Mode Decomposition, extracts electrical features, and runs machine learning inference.

### Recommended Directory Structure

```
nilm_ml_service/
├── config.py                  # Environment configurations & MQTT settings
├── main.py                    # Service entry point & MQTT event loop
├── mqtt/
│   ├── __init__.py
│   ├── subscriber.py          # Waveform subscriber handling
│   └── publisher.py           # Result publisher handling
├── dsp/
│   ├── __init__.py
│   ├── normalization.py       # Z-score & Min-Max scaling
│   ├── vmd_engine.py          # VMD decomposition wrapper (vmdpy)
│   └── feature_extraction.py  # FFT, P, Q, PF, THD, & transient metrics
├── ml/
│   ├── __init__.py
│   ├── tflite_engine.py       # TFLite Runtime inference pipeline
│   └── models/
│       └── nilm_model_v1.tflite # Quantized model file
└── utils/
    └── logger.py              # Centralized logging module
```

### Python Data Processing Pipeline

```
  [ Incoming Waveform Packet ]
              |
              v
  [ Signal Normalization ] ---> (Scale V & I arrays to zero mean, unit variance)
              |
              v
  [ VMD Decomposition ]   ---> (Extract K Intrinsic Mode Functions: IMF1, IMF2, ..., IMFk)
              |
              v
  [ IMF Selection ]       ---> (Filter out high-frequency noise & isolate fundamental modes)
              |
              v
  [ Feature Extraction ]  ---> (Compute RMS, P, Q, PF, THD, Harmonics & Transient Features)
              |
              v
  [ TFLite ML Model ]     ---> (Predict Appliance Classes, States & Power Distro)
              |
              v
  [ Publish Results ]     ---> (Format JSON & Publish to `nilm/device01/result`)
```

---

## 8. Variational Mode Decomposition (VMD) Integration

Variational Mode Decomposition (VMD) is an adaptive, non-recursive signal decomposition technique. It decomposes a complex multi-component signal $x(t)$ into a discrete number of sub-signals (Intrinsic Mode Functions, or IMFs), each having specific sparsity properties and centered around a specific central frequency $\omega_k$.

```
+-------------------------------------------------------------------------+
| RAW OVERLAPPING CURRENT WAVEFORM (Total Load Signal)                    |
+-------------------------------------------------------------------------+
                                    |
                                    v Executing VMD (K = 4 Modes)
+-------------------------------------------------------------------------+
| IMF 1: Fundamental Frequency Component (50 Hz / Main Power Line Draw)    |
+-------------------------------------------------------------------------+
| IMF 2: Low-Order Harmonics (150 Hz / 3rd Harmonic - Non-linear Loads)   |
+-------------------------------------------------------------------------+
| IMF 3: High-Order Switching Harmonics (SMPS / Laptop Charger / TV)      |
+-------------------------------------------------------------------------+
| IMF 4: High-Frequency Transients & Inrush Noise                         |
+-------------------------------------------------------------------------+
```

### Key VMD Clarifications

* **VMD is Preprocessing**: VMD is a **signal separation and noise reduction tool**, not a classifier. It breaks down complex overlapping signals into component sub-bands.
* **Why VMD over EMD**: Traditional Empirical Mode Decomposition (EMD) suffers from mode mixing and is sensitive to noise. VMD uses an alternate direction method of multipliers (ADMM) optimization that is mathematically robust against mode mixing.

---

## 9. VMD + FFT Complementary Feature Extraction

Using VMD does **not** replace Fast Fourier Transform (FFT). Instead, VMD acts as a decomposition front-end, and FFT/statistical analysis are performed on individual IMFs to construct a comprehensive feature vector for classification.

```
Raw Signal ---> VMD ---> [ IMF 1 ] ---> FFT / P / Q / THD ----+
                         [ IMF 2 ] ---> Harmonic Spectrum -----+---> Combined Feature Vector
                         [ IMF 3 ] ---> Transient Envelope ----+          |
                                                                          v
                                                                 [ TFLite Classifier ]
```

### Complete Feature Set Matrix

1. **Fundamental Electrical Features**:
   * Active Real Power ($P = \frac{1}{N}\sum v[n] \cdot i[n]$)
   * Reactive Power ($Q = \sqrt{S^2 - P^2}$)
   * Apparent Power ($S = V_{rms} \times I_{rms}$)
   * Power Factor ($PF = \frac{P}{S}$)
2. **Harmonic Features (via FFT on IMFs)**:
   * Magnitudes and phase angles of $3^{rd}, 5^{th}, 7^{th}, \text{ and } 9^{th}$ harmonics.
   * Total Harmonic Distortion ($THD_I = \frac{\sqrt{\sum_{h=2}^{\infty} I_h^2}}{I_1}$).
3. **Statistical Waveform Features**:
   * Peak-to-Peak Amplitude, Crest Factor ($\frac{I_{peak}}{I_{rms}}$), Form Factor, Skewness, Kurtosis.
4. **Transient & Switching Features**:
   * Startup current surge peak ($I_{surge}$), turn-ON $dI/dt$ slope, transient decay duration.

---

## 10. Machine Learning Pipeline & TFLite Model

```
[ Data Collection ] ---> [ Feature Scaling ] ---> [ Model Training ] ---> [ Quantization ] ---> [ TFLite Deployment ]
(Labeled Dataset)       (StandardScaler)         (Deep Neural Net / RF)   (Float32 -> INT8)     (Python ML Service)
```

### Supported Appliance Classes

1. **Refrigerator**: Cyclical compressor, inductive power factor, moderate starting transient (~150W).
2. **Geyser / Water Heater**: Purely resistive, high power (~2000W), unit power factor ($PF \approx 1.0$).
3. **Ceiling Fan**: Low-power inductive load (~60W–75W), steady-state draw.
4. **Electric Kettle**: High resistive load (~1500W), sharp ON/OFF state transition.
5. **Clothes Iron**: High resistive load (~1000W) with periodic thermostatic cycling.
6. **Laptop Charger**: Low-power Switched-Mode Power Supply (SMPS), rich in $3^{rd}$ and $5^{th}$ harmonics.
7. **Television**: Medium-power SMPS (~100W), distinct harmonic signature.
8. **Air Conditioner**: High inductive load with variable-frequency inverter transient signatures.
9. **Other / Unknown**: Fallback classification for unrecognized electrical signatures.

### Model Outputs

For each window, the TFLite model produces:
* **Appliance Class Identification** (Multi-label probabilities).
* **State Confidence Score** (0.0 to 1.0 / 0% to 100%).
* **Estimated Disaggregated Power (Watts)** per detected appliance.

---

## 11. Node-RED IoT Orchestration & Alert Engine

Node-RED serves as the rule evaluation engine, event coordinator, and database writer. It does **not** execute heavy matrix math or VMD.

```
 [ MQTT Broker ] ---(`nilm/device01/result`)---> [ Node-RED ]
                                                      |
         +--------------------------------------------+--------------------------------------------+
         |                                            |                                            |
         v                                            v                                            v
 [ InfluxDB Writer Node ]                    [ High-Power Rule ]                         [ Left-ON Rule ]
 (Writes energy telemetry)                (Triggers if total P > 4kW)                (Triggers if Geyser ON > 45m)
         |                                            |                                            |
         v                                            +---------------------+----------------------+
 [ InfluxDB Storage ]                                                       |
                                                                            v
                                                                   [ MQTT Alert Topic ]
                                                                (`nilm/device01/alert`)
```

### Rule & Alert Logic Examples

1. **High-Power Overload Alert**:
   * *Condition*: `total_real_power_w > 4000W` continuously for > 10 seconds.
   * *Action*: Generate critical safety alert payload to `nilm/device01/alert`.
2. **Appliance Left-ON Warning**:
   * *Condition*: `appliance == "Geyser"` AND `state == "ON"` for > 45 consecutive minutes.
   * *Action*: Trigger "Geyser Left ON" notification.
3. **Phantom Load Detection**:
   * *Condition*: Total power during 1:00 AM–5:00 AM window remains > 150W continuously.
   * *Action*: Highlight standby power waste in daily dashboard summary.

---

## 12. InfluxDB Time-Series Data Architecture

InfluxDB is optimized for high-write-throughput timestamped data, downsampling, and continuous aggregation over time ranges.

```
Measurement: appliance_telemetry
+------------------+------------------+---------------+----------------+----------------+---------------+-----------------------+
| timestamp (time) | device_id (tag)  | appliance     | power_w        | energy_kwh     | confidence    | state (tag)           |
|                  |                  | (tag)         | (field)        | (field)        | (field)       |                       |
+------------------+------------------+---------------+----------------+----------------+---------------+-----------------------+
| 1776595200000    | esp32_sensor_01  | Geyser        | 1800.0         | 0.45           | 0.96          | ON                    |
| 1776595200000    | esp32_sensor_01  | Fan           | 60.0           | 0.015          | 0.91          | ON                    |
+------------------+------------------+---------------+----------------+----------------+---------------+-----------------------+
```

### Data Schema Definition

* **Measurement**: `appliance_telemetry`
* **Tag Set** (Indexed for fast query filtering):
  * `device_id`: Identifier of the physical ESP32 sensor node.
  * `appliance`: Name of the disaggregated appliance.
  * `state`: Operating state (`ON`, `OFF`, `STANDBY`).
* **Field Set** (Unindexed values):
  * `power_w` (Float): Instantaneous active real power in Watts.
  * `energy_kwh` (Float): Accumulated energy consumption in kWh.
  * `confidence` (Float): Classification probability score (0.0 to 1.0).
* **Timestamp**: RFC3339 nanosecond precision timestamp.

---

## 13. FastAPI Backend Service

To ensure security and decouple data storage from presentation, the React application **never queries InfluxDB directly**. All queries route through the FastAPI middleware layer.

```
 [ React Frontend UI ] <--- HTTP REST / JSON ---> [ FastAPI Gateway ] <--- Flux Queries ---> [ InfluxDB ]
```

### Core API Endpoints

| Method | Endpoint Path | Description / Query Parameters | Response Format |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/v1/dashboard` | Overall system state, total live power, active device count. | `JSON` |
| `GET` | `/api/v1/appliances` | Instantaneous status and power consumption of all appliances. | `JSON` |
| `GET` | `/api/v1/energy/today` | Cumulative kWh consumption per appliance for today. | `JSON` |
| `GET` | `/api/v1/energy/history` | Historical energy data (`?start=2026-08-01&end=2026-08-19&interval=1h`). | `JSON` |
| `GET` | `/api/v1/alerts` | List of recent safety and energy alerts (`?limit=20`). | `JSON` |
| `GET` | `/api/v1/appliance/{name}`| Detailed historical metrics for a specific named appliance. | `JSON` |

---

## 14. Real-Time Telemetry via WebSockets

While REST APIs handle historical data retrieval, live telemetry feeds rely on **WebSockets** to deliver sub-second UI updates without polling overhead.

```
 [ Python ML Result ] ---> [ MQTT ] ---> [ FastAPI WebSocket Hub ] === WebSocket Stream ===> [ React Dashboard UI ]
                                                                   (ws://localhost:8000/ws)
```

### WebSocket Event Data Flow

1. Python ML Service publishes disaggregation output to `nilm/device01/result`.
2. FastAPI background worker receives the MQTT message.
3. FastAPI WebSocket Broadcast Manager pushes the updated JSON payload over active client socket connections (`ws://...`).
4. React frontend context receives the payload and updates the dashboard state instantly without requiring a full page refresh.

---

## 15. React Dashboard UI Blueprint

The React frontend presents real-time power metrics, appliance states, historical energy charts, and active alerts.

### Interactive UI Component Layout Mockup

```
+-----------------------------------------------------------------------------------------+
|  NILM SMART ENERGY MONITORING DASHBOARD                          [ Node: ESP32_01 (ONLINE) ]|
+-----------------------------------------------------------------------------------------+
| [ INSTANT POWER ]      [ TODAY'S CONSUMPTION ]   [ ESTIMATED COST ]    [ ACTIVE APPLIANCES ]|
|    1,860.5 W               4.28 kWh                 $ 0.64                    2 / 8     |
+-----------------------------------------------------------------------------------------+
| REAL-TIME POWER CONSUMPTION (LIVE WEBSOCKET STREAM)                                     |
|  2500W |                                                                                |
|  2000W |                       +----------------------------- (Geyser Turned ON)        |
|  1500W |                       |                                                        |
|  1000W |                       |                                                        |
|   500W |-----------------------+                                                        |
|     0W +------------------------------------------------------------------------------  |
|        15:00     15:02     15:04     15:06     15:08     15:10     15:12     15:14      |
+--------------------------------------------+--------------------------------------------+
| DISAGGREGATED APPLIANCE STATUS             | APPLIANCE ENERGY BREAKDOWN (TODAY)         |
|  [●] Geyser       1800 W (96% Conf) [ON ]  |                                            |
|  [●] Fan            60.5 W (91% Conf) [ON ]  |    Geyser:  ████████████████ 82% (3.51 kWh) |
|  [○] Refrigerator    0 W (---- Conf) [OFF]  |    Fan:     ███ 12% (0.51 kWh)           |
|  [○] Elec. Kettle    0 W (---- Conf) [OFF]  |    Other:   █ 6% (0.26 kWh)              |
+--------------------------------------------+--------------------------------------------+
| RECENT SYSTEM ALERTS & NOTIFICATIONS                                                    |
|  [⚠ WARNING] 15:05:12 - High Power Load Detected (>1800W)                               |
|  [ℹ INFO]    14:30:00 - Refrigerator Compressor Completed Cycle                         |
+-----------------------------------------------------------------------------------------+
```

---

## 16. Deployment Option A — Per-Circuit Monitoring (Primary Prototype Baseline)

Per-circuit monitoring isolates individual appliance lines or small dedicated extension sockets for prototype development and initial ML model training.

```
 +-----------------------------+
 | Multi-Socket Extension Board|
 +--------------+--------------+
                |
                v Single Appliance Line (e.g., Kettle / Geyser)
      +---------+---------+
      |  SCT-013 CT Clamp |
      +---------+---------+
                |
                v Clean, Unmixed Signal
      +---------+---------+
      |    ESP32 DAQ      |
      +-------------------+
```

### Why Per-Circuit is the Primary Prototype Target

* **Clean Signal Acquisition**: Captures isolated current waveforms without multi-appliance overlap.
* **Ground Truth Validation**: Provides labeled training data for initial ML model training.
* **Low Initial Complexity**: Allows validation of the ESP32 → MQTT → VMD → ML pipeline before attempting complex multi-label signal disaggregation.

---

## 17. Deployment Option B — Whole-Panel Monitoring (Advanced Research Scope)

Whole-panel monitoring places a single CT clamp on the main breaker line of a home or distribution box.

```
 +---------------------------------------------------------+
 | Main Distribution Box (MCB Main Live Feed Wire)         |
 +----------------------------+----------------------------+
                              |
                              v Single Central Sensing Point
                    +---------+---------+
                    |  SCT-013 CT Clamp |
                    +---------+---------+
                              |
                              v Aggregated Mixed Waveform
                    +---------+---------+
                    |    ESP32 DAQ      |
                    +-------------------+
```

### Technical Challenges of Whole-Panel NILM

* **Overlapping Electrical Signatures**: Simultaneous operation of high-power loads (e.g., 2000W geyser) can mask low-power loads (e.g., 15W charger).
* **Non-Stationary Noise**: Switching transients from various household electronics introduce broadband line noise.
* **Multi-Label Disaggregation**: Requires VMD signal decomposition and advanced multi-label classifiers to isolate simultaneous appliance states from a single mixed waveform.

---

## 18. Multi-Appliance Disaggregation Strategies

To disaggregate overlapping signatures during whole-panel monitoring, VMD is combined with complementary electrical analysis techniques:

```
 Aggregated Waveform
         |
         v
 [ Event Detection ] ---> (Detect Step Changes: ΔP > Threshold or ΔQ > Threshold)
         |
         v
 [ Window Isolation ] ---> (Isolate Transient Switching Region: ±500 ms)
         |
         v
 [ VMD Decomposition ] ---> (Decompose Transient Window into Band-Limited IMFs)
         |
         v
 [ Feature Extraction ] ---> (Extract ΔP, ΔQ, V-I Trajectory & Harmonic Spectrum)
         |
         v
 [ TFLite Classifier ] ---> (Identify Newly Switched Appliance Class)
         |
         v
 [ State Tracking FSM ] ---> (Update Global Appliance State Table: ON/OFF)
```

### Complementary Disaggregation Techniques

1. **Active & Reactive Power ($P-Q$) Plane Trajectory**: Maps operational state changes as vectors in two-dimensional $P-Q$ space. Different appliances occupy distinct regions based on their electrical characteristics.
2. **Voltage-Current ($V-I$) Trajectory Analysis**: Plotting instantaneous current against voltage over a full cycle creates closed Lissajous patterns. The geometrical shape, curvature, and phase area serve as a diagnostic signature for appliance type.
3. **Event-Based Switching Detection**: Continuously monitors the total power curve for sharp step changes ($\Delta P$, $\Delta Q$). When a step change exceeds a threshold, an event window is isolated for VMD feature extraction and classification.
4. **Finite State Machine (FSM) State Tracking**: Maintains an active inventory of known household appliance operational states (`OFF` $\rightarrow$ `STARTUP` $\rightarrow$ `STEADY_RUN` $\rightarrow$ `TURNOFF`).

---

## 19. Clarification of Technical Roles

A common misconception is that VMD handles the entire disaggregation process independently. The table below delineates the distinct role of each subsystem:

```
+----------------------------------------------------------------------------------+
|               NILM SIGNAL DISAGGREGATION PROCESSING PIPELINE                     |
|                                                                                  |
|  [ Raw Waveform ] ---> (1) VMD          : Signal Decomposition Front-End        |
|                   ---> (2) FFT / Features: Feature Extraction & Metrics           |
|                   ---> (3) TFLite ML     : Machine Learning Classification       |
|                   ---> (4) Node-RED      : State Tracking & Alert Logic          |
+----------------------------------------------------------------------------------+
```

| Subsystem / Technique | Exact Technical Role | What It Does NOT Do |
| :--- | :--- | :--- |
| **VMD (Variational Mode Decomposition)** | **Signal Decomposition Front-End**: Decomposes raw current waveforms into sub-band Intrinsic Mode Functions (IMFs). | Does **not** classify appliances, calculate power metrics, or track ON/OFF states. |
| **FFT / Feature Extraction** | **Feature Representation**: Computes harmonics, RMS values, $P$, $Q$, $PF$, and $THD$ from raw IMFs. | Does **not** perform classification by itself or store historical data. |
| **TFLite ML Engine** | **Pattern Classification**: Maps feature vectors to appliance IDs and predicts power draw. | Does **not** clean raw noise or evaluate business alert rules directly. |
| **Node-RED Logic** | **State Tracking & Orchestration**: Evaluates duration rules, triggers alerts, formats DB writes. | Does **not** run matrix math, continuous DSP algorithms, or model training. |

---

## 20. Local Development Environment Setup

For initial prototyping and testing, the entire software infrastructure runs locally on a single developer laptop. The ESP32 connects wirelessly over local Wi-Fi.

```
+-----------------------------------------------------------------------------------------+
| LOCAL DEVELOPER LAPTOP / WORKSTATION                                                    |
|                                                                                         |
|  +-----------------------------------------------------------------------------------+  |
|  | DOCKER COMPOSE CONTAINER NETWORK                                                  |  |
|  |                                                                                   |  |
|  |  [ Mosquitto MQTT ] <---> [ Python ML Service ] <---> [ Node-RED Orchestrator ]   |  |
|  |  (Port 1883)               (Local Background Process)  (Port 1880)              |  |
|  |                                                              |                    |  |
|  |                                                              v                    |  |
|  |  [ React + Vite UI ] <---> [ FastAPI Gateway ] <------ [ InfluxDB Storage ]       |  |
|  |  (Port 5173)               (Port 8000)                 (Port 8086)              |  |
|  +-----------------------------------------------------------------------------------+  |
+-------------------------------------------^---------------------------------------------+
                                            | Local Wi-Fi (MQTT Telemetry)
                                  +---------+---------+
                                  |    ESP32 DAQ      |
                                  +-------------------+
```

### Docker Compose Service Definition (`docker-compose.yml`)

```yaml
version: '3.8'

services:
  mosquitto:
    image: eclipse-mosquitto:2.0
    container_name: nilm_mosquitto
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config

  influxdb:
    image: influxdb:2.7
    container_name: nilm_influxdb
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=adminpassword123
      - DOCKER_INFLUXDB_INIT_ORG=nilm_org
      - DOCKER_INFLUXDB_INIT_BUCKET=energy_telemetry

  nodered:
    image: nodered/node-red:latest
    container_name: nilm_nodered
    ports:
      - "1880:1880"
    depends_on:
      - mosquitto
      - influxdb

  fastapi:
    build: ./backend
    container_name: nilm_fastapi
    ports:
      - "8000:8000"
    depends_on:
      - influxdb

  frontend:
    build: ./frontend
    container_name: nilm_react_ui
    ports:
      - "5173:5173"
    depends_on:
      - fastapi
```

---

## 21. Sequential Development Milestones

To ensure progress and structured debugging, development follows a milestone roadmap. Each phase must be fully validated before moving to the next.

```
 [ M1: Sensors ] -> [ M2: ESP32-MQTT ] -> [ M3: Python VMD ] -> [ M4: TFLite ML ] -> [ M5: E2E MQTT ] -> [ M6: Storage ] -> [ M7: Dashboard ]
```

### Milestone Breakdown

#### Milestone 1: Sensor Calibration & Signal Validation
* **Goal**: Validate analog voltage and current readings from physical hardware sensors.
* **Implementation**: Wire SCT-013 and ZMPT101B to ESP32 ADC pins. Write basic calibration sketch using Arduino IDE.
* **Expected Output**: Stable, noise-free 50 Hz sine wave readings visible on Arduino Serial Plotter.
* **Verification Test**: Verify measured AC voltage against a reference digital multimeter.
* **Prerequisites**: Functional hardware components and breadboard setup.

#### Milestone 2: ESP32 High-Speed Sampling & MQTT Transmission
* **Goal**: Achieve synchronous 8–10 kHz sampling and transmit waveform buffer windows over Wi-Fi.
* **Implementation**: Implement microsecond hardware timer loops, buffer windowing, and MQTT client publishing.
* **Expected Output**: ESP32 publishes intact 512/1024 sample payload arrays to `nilm/device01/waveform`.
* **Verification Test**: Inspect incoming JSON packet frequency and integrity using MQTT Explorer.
* **Prerequisites**: Successful completion of Milestone 1; operational Mosquitto broker.

#### Milestone 3: Python VMD Signal Processing Pipeline
* **Goal**: Ingest raw MQTT waveforms in Python and execute VMD decomposition.
* **Implementation**: Develop Python subscriber service, integrate `vmdpy` library, and implement feature extraction logic.
* **Expected Output**: Successful extraction of $K$ IMFs and computed feature vectors ($P, Q, PF, THD$).
* **Verification Test**: Plot raw signals alongside decomposed IMFs using `matplotlib` to verify mode separation.
* **Prerequisites**: Successful completion of Milestone 2.

#### Milestone 4: ML Dataset Collection, Model Training & TFLite Deployment
* **Goal**: Train a classification model for initial target appliances and export it to TFLite format.
* **Implementation**: Record feature vectors for target loads, train classifier in PyTorch/TensorFlow, apply INT8 quantization, and export `.tflite` model.
* **Expected Output**: A lightweight TFLite model file capable of running sub-10ms inference in Python.
* **Verification Test**: Evaluate test set classification accuracy (target $>90\%$).
* **Prerequisites**: Successful completion of Milestone 3.

#### Milestone 5: End-to-End MQTT Telemetry Integration
* **Goal**: Link Python ML predictions back into the MQTT ecosystem.
* **Implementation**: Configure Python ML service to publish output payloads to `nilm/device01/result`.
* **Expected Output**: Structured disaggregation JSON payloads published continuously to the broker.
* **Verification Test**: Subscribe to `nilm/device01/result` and confirm live prediction updates when toggling appliances.
* **Prerequisites**: Successful completion of Milestone 4.

#### Milestone 6: Node-RED Orchestration, Alerting & InfluxDB Persistence
* **Goal**: Automate data persistence and configure energy alert rules.
* **Implementation**: Create Node-RED flow to parse prediction results, evaluate threshold logic, and write records to InfluxDB.
* **Expected Output**: Time-series records stored in InfluxDB; test alerts published to `nilm/device01/alert`.
* **Verification Test**: Query InfluxDB via its administrative UI to confirm data structure and retention.
* **Prerequisites**: Successful completion of Milestone 5.

#### Milestone 7: FastAPI Gateway & React Real-Time Dashboard Integration
* **Goal**: Deliver a user-facing dashboard displaying real-time metrics and historical analytics.
* **Implementation**: Develop FastAPI REST/WebSocket endpoints; construct React frontend with interactive Recharts components.
* **Expected Output**: Fully functional web dashboard displaying live power curves, appliance state cards, and historical energy charts.
* **Verification Test**: Toggle a physical load and observe sub-second UI updates on the web dashboard.
* **Prerequisites**: Successful completion of Milestone 6.

---

## 22. End-to-End Real-World Data Journey

This section traces the path of a single electrical signal from appliance activation to dashboard rendering.

```
 [ Appliance Turned ON ]
            | (Physical Current Flow)
            v
 [ SCT-013 & ZMPT101B Sensors ]
            | (Induced Low-Voltage Analog Signal: 0-3.3V)
            v
 [ ESP32 Dual ADC Inputs ]
            | (Synchronous Sampling @ 8-10 kHz -> Accumulated into 1024-sample Ring Buffer)
            v
 [ ESP32 Wi-Fi Transmission ]
            | (MQTT JSON/Binary Packet published to `nilm/device01/waveform`)
            v
 [ Mosquitto MQTT Broker ]
            | (Routes packet to subscriber)
            v
 [ Python ML Service ]
            |--> Signal Normalization (Z-score Scaling)
            |--> VMD Decomposition (Separates signal into K=4 IMFs via ADMM)
            |--> Feature Extraction (Calculates Harmonics, P, Q, PF, THD, Transient Envelopes)
            |--> TFLite Inference Engine (Predicts Appliance Classes, Power Draw, Confidence)
            v
 [ Python Result Transmission ]
            | (Publishes prediction payload to `nilm/device01/result`)
            v
 [ Mosquitto MQTT Broker ]
            | (Routes prediction result)
            v
 [ Node-RED Orchestrator ]
            |--> Evaluates Safety Rules (Overload / Left-ON Warnings)
            |--> Formats Time-Series Record & Writes to InfluxDB Storage
            v
 [ FastAPI Web Gateway ]
            | (Pulls live MQTT result & broadcasts via WebSocket `ws://...`)
            v
 [ React UI Dashboard ]
            | (Parses WebSocket JSON payload & updates UI state in sub-second time)
            v
 [ End-User Interface Updated ] (Live Power Gauge, Appliance State Card & Alert Banner)
```

---

## 23. Resolution of Primary Problem Statements

Our proposed architecture directly addresses the three key problem statements:

```
  PROBLEM STATEMENT 1: Opaque Electricity Bills
  └── SOLUTION: Real-time disaggregation & InfluxDB history provide appliance-level cost breakdowns.

  PROBLEM STATEMENT 2: High-Power Appliances Left ON
  └── SOLUTION: Node-RED duration/threshold rules trigger automated high-priority alerts.

  PROBLEM STATEMENT 3: Expensive Per-Appliance Smart Plugs
  └── SOLUTION: Single-point sensing at the main line combined with NILM software disaggregation.
```

1. **Problem 1: Opaque Electricity Bills**
   * *Resolution*: By breaking down total consumption into individual appliance contributions, the system provides transparent, itemized energy costs (e.g., "Geyser cost $12.50 this month").
2. **Problem 2: Appliances Left ON Unattended**
   * *Resolution*: Node-RED evaluates continuous operating duration against safety thresholds (e.g., Geyser ON for $>45$ min). When a rule is violated, an instant notification is dispatched to the user dashboard.
3. **Problem 3: High Cost of Smart-Home Hardware**
   * *Resolution*: Rather than installing expensive smart plugs on every wall outlet, a single central CT clamp and voltage sensor monitor the entire panel, reducing hardware deployment costs.

---

## 24. Final Technology Architecture & Team Interface Contract

```
 HARDWARE EDGE LAYER
 [ ESP32 + SCT-013 + ZMPT101B ]
               |
               v (Wi-Fi / MQTT: `nilm/device01/waveform`)
 BROKER LAYER
 [ MOSQUITTO MQTT BROKER ]
               |
               v (MQTT Subscriber)
 SIGNAL PROCESSING & ML LAYER
 [ PYTHON ML SERVICE ]
   ├── Signal Normalization
   ├── Variational Mode Decomposition (VMD)
   ├── Feature Extraction (FFT, P, Q, PF, THD)
   └── TFLite Inference Execution
               |
               v (MQTT Publisher: `nilm/device01/result`)
 ORCHESTRATION & STORAGE LAYER
 [ NODE-RED ORCHESTRATOR ]
   ├── Evaluate Alert Rules
   └── Write Telemetry to Database
               |
               v (Flux Protocol)
 [ INFLUXDB TIME-SERIES DB ]
               |
               v (REST API / Flux Queries)
 BACKEND API GATEWAY
 [ FASTAPI BACKEND SERVICE ] <---> [ WEBSOCKET SERVER ]
                                          |
                                          v (HTTP / WebSocket Stream)
 FRONTEND USER INTERFACE
 [ REACT + VITE DASHBOARD UI ]
```

### Team Interface Contract Table

To enable parallel development across team members, the system boundaries and data exchange contracts are defined below:

| Interface Boundary | Source Component | Destination Component | Protocol / Format | Data Payload Schema | Module Lead |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sensor $\rightarrow$ ESP32** | Physical Sensors | ESP32 Hardware | Analog Voltages (0–3.3V) | Biased AC sine waves (V on GPIO 35, I on GPIO 34). | Hardware Lead |
| **ESP32 $\rightarrow$ Broker** | ESP32 Firmware | Mosquitto Broker | MQTT over Wi-Fi | Topic: `nilm/device01/waveform`<br>Payload: Waveform JSON array window. | Embedded Lead |
| **Broker $\rightarrow$ Python** | Mosquitto Broker | Python ML Service | MQTT Subscription | Subscribes to raw waveform topic. | ML/DSP Lead |
| **Python $\rightarrow$ Broker** | Python ML Service | Mosquitto Broker | MQTT Publishing | Topic: `nilm/device01/result`<br>Payload: Appliance prediction JSON. | ML/DSP Lead |
| **Broker $\rightarrow$ Node-RED** | Mosquitto Broker | Node-RED Engine | MQTT Subscription | Subscribes to disaggregation results. | Backend Lead |
| **Node-RED $\rightarrow$ DB** | Node-RED Engine | InfluxDB Storage | Influx Line Protocol | Measurement: `appliance_telemetry`<br>Tags: `device_id`, `appliance`. | Backend Lead |
| **DB $\rightarrow$ FastAPI** | InfluxDB Storage | FastAPI Gateway | Flux Query Language | Structured time-series aggregation queries. | Backend Lead |
| **FastAPI $\rightarrow$ React** | FastAPI Gateway | React Frontend UI | REST & WebSockets | REST JSON responses & real-time WebSocket stream. | Frontend Lead |

---

## 25. Final Recommendation & Baseline Freeze Strategy

For the major project implementation, the team must adhere strictly to the following baseline freeze strategy:

### Primary Baseline (Prototype 1 Focus)

1. **Deployment Architecture**: **Per-Circuit Monitoring** (Deployment Option A). Measure isolated loads first to build high-quality dataset signatures.
2. **Edge Hardware Role**: ESP32 functions strictly as a high-speed **Data Acquisition Unit (DAQ)**. No VMD or ML execution on the microcontroller initially.
3. **Core Processing**: Run Python ML Service locally on a developer workstation consuming MQTT telemetry.
4. **Target Appliance Set**: Initial focus on **5 distinct loads**: Refrigerator, Geyser, Ceiling Fan, Electric Kettle, and Laptop Charger.

### Advanced Research Scope (Phase 2 Expansion)

1. **Whole-Panel Monitoring**: Transition to single-point monitoring at the main breaker once single-load signatures are validated.
2. **Edge Optimization Studies**: Profile Python service memory, CPU footprint, and latency. Evaluate feasibility of porting lightweight TFLite inference or simplified feature extraction to ESP32 hardware in future iterations.

---

*This document represents the finalized technical reference for the NILM Smart Energy Monitor project. All team members should follow the specifications and interface contracts detailed above.*
