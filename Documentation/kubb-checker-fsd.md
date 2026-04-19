# Kubb-Checker — Functional Specification Document (FSD)

**Version:** 0.1  
**Date:** 2026-04-19  
**Status:** Draft  
**Author:** rolf@dubitzky.de

---

## 1. System Overview

### 1.1 Purpose

Kubb-Checker assists referees in a game of Kubb by automatically determining whether a throw is valid. Under official Kubb rules, a thrown baton must rotate at least 360° around its minor axis (end-over-end) between leaving the thrower's hand and hitting a Kubba (wooden block). Kubb-Checker measures this rotation using inertial sensors embedded in the batons and presents the verdict on a graphical web interface visible to all players.

### 1.2 Problem Statement

Validating throws by visual inspection is subjective and error-prone, especially at high throwing speeds. Kubb-Checker provides an objective, sensor-based validation that eliminates disputes and enhances the fairness of the game.

### 1.3 Users / Stakeholders

| Stakeholder | Role |
|-------------|------|
| Referee / Game Operator | Monitors the web interface to validate throws; manages device names, calibration, and statistics |
| Players | Observe throw validity results on the web interface |
| System Administrator | Handles firmware updates and device provisioning |

### 1.4 Goals & Non-Goals

**Goals:**
- Measure and record baton rotation and flight parameters for each throw
- Display real-time throw validity on a mobile-accessible web interface
- Support at least 6 simultaneous baton sensors in a single game
- Support wireless firmware updates for all devices

**Non-Goals:**
- Automated game scoring or full game management
- Cloud connectivity or internet-based data sync
- Audio or visual feedback on the baton itself (LEDs, buzzers)
- Support for multiple simultaneous games

### 1.5 High-Level System Flow

```
[Baton thrown]
     │
     ▼
[Sensor: Detect throw start (IMU threshold)]
     │
     ▼
[Sensor: Integrate gyro data during flight]
     │
     ▼
[Sensor: Detect landing (impact spike)]
     │
     ▼
[Sensor: Compute rotation, time-of-flight, max accel → send via ESP-NOW]
     │
     ▼
[Hub: Receive, store, and classify throw (valid / invalid)]
     │
     ▼
[Web Interface: Display result to all connected mobile clients]
```

---

## 2. System Architecture

### 2.1 Logical Architecture

The system consists of two logical subsystems:

**Kubb-Checker-Sensor** (embedded in each baton):
- IMU data acquisition and integration
- Flight event detection (throw start / landing)
- Throw analysis and result computation
- Battery monitoring
- ESP-NOW client (star topology node)

**Kubb-Checker-Hub** (central device):
- ESP-NOW server (star topology center)
- Data aggregation and game statistics
- WiFi Access Point host
- Embedded HTTP web server
- OTA coordinator

```
┌──────────────────────────────────────────────────────────────┐
│                       Mobile Devices                         │
│                (browsers via WiFi 802.11 b/g/n)              │
└─────────────────────────┬────────────────────────────────────┘
                          │ HTTP (polling or WebSocket)
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                    Kubb-Checker-Hub                          │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │  ESP-NOW RX   │  │  Data Store   │  │  HTTP Server    │  │
│  │  (star hub)   │→ │  (RAM / NVS)  │→ │  (web UI + API) │  │
│  └───────────────┘  └───────────────┘  └─────────────────┘  │
└──────────────────────────┬───────────────────────────────────┘
                           │ ESP-NOW (2.4 GHz)
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │  Sensor 1    │ │  Sensor 2    │ │  Sensor N≥6  │
  │ ESP32-C3 +   │ │ ESP32-C3 +   │ │ ESP32-C3 +   │
  │ BMI160 + bat │ │ BMI160 + bat │ │ BMI160 + bat │
  └──────────────┘ └──────────────┘ └──────────────┘
```

### 2.2 Hardware / Platform Architecture

#### Kubb-Checker-Sensor

| Attribute | Value |
|-----------|-------|
| MCU | ESP32-C3 |
| IMU | Bosch BMI160 (gyroscope + accelerometer) |
| IMU Interface | I2C |
| Power | LiIon 1S cell (3.7 V nominal) |
| Form Factor | Inside wooden baton (~300 mm × ~40 mm diameter) |
| Quantity | ≥ 6 per game |

**Axis Convention:**
- **Major axis**: Along the baton's length (longitudinal)
- **Minor axis**: Perpendicular to the baton's length (transverse)
- A valid throw requires ≥ 360° rotation around the **minor axis** (end-over-end tumbling)

The BMI160 shall be mounted so that one of its gyroscope axes aligns with the baton's minor axis. The axis mapping (which BMI160 axis corresponds to major vs. minor) shall be defined as a compile-time constant and documented in the hardware bring-up notes.

#### Kubb-Checker-Hub

| Attribute | Value |
|-----------|-------|
| MCU | ESP32 (variant TBD; must support simultaneous ESP-NOW + WiFi AP) |
| Storage | NVS flash (sensor registry, configuration) |
| Connectivity | ESP-NOW toward sensors; WiFi AP toward mobile clients |
| Power | External supply — USB power bank or mains adapter (assumed) |

### 2.3 Software Architecture

#### Sensor Firmware Tasks

| Task | Priority | Description |
|------|----------|-------------|
| `imu_task` | High | I2C polling, BMI160 data acquisition at ≥ 400 Hz |
| `flight_detect_task` | High | State machine: Still → Moving → Flying → Still |
| `analysis_task` | Medium | Integrate gyro data, compute throw result when flight ends |
| `espnow_tx_task` | Medium | Send status and throw result packets to hub |
| `battery_task` | Low | Periodic ADC read of battery voltage → SoC estimate |
| `serial_debug_task` | Low | Format and output structured debug info via UART |

**Sensor Boot Sequence:**
1. Initialize UART / serial logging; output firmware version
2. Initialize I2C bus and BMI160; verify chip ID
3. Run gyroscope zero-rate offset calibration (sensor at rest, ≥ 3 s)
4. Load device name from NVS (or use MAC-derived default)
5. Initialize ESP-NOW; register hub MAC as unicast peer
6. Start all FreeRTOS tasks
7. Enter normal operation (Still state)

#### Hub Firmware Tasks

| Task | Priority | Description |
|------|----------|-------------|
| `espnow_rx_task` | High | Receive and dispatch packets from sensors |
| `data_store_task` | Medium | Update sensor state / throw records and statistics |
| `wifi_ap_task` | Medium | Maintain WiFi AP; manage connected clients |
| `http_server_task` | Medium | Serve web UI and REST API endpoints |
| `serial_debug_task` | Low | Output received data and statistics via UART |

**Hub Boot Sequence:**
1. Initialize UART / serial logging; output firmware version
2. Initialize NVS; load sensor registry (names keyed by MAC) and configuration
3. Initialize WiFi in AP+STA mode (required for concurrent ESP-NOW + AP)
4. Start WiFi Access Point
5. Initialize ESP-NOW
6. Start HTTP server
7. Start all FreeRTOS tasks
8. Enter normal operation

**OTA Update Model:**

- **Sensor OTA**: Web UI trigger → Hub sends `CMD_OTA_START` via ESP-NOW → Sensor suspends ESP-NOW and connects to hub WiFi AP as STA → Sensor downloads `sensor.bin` via HTTP from hub → Sensor verifies integrity, writes to OTA partition B, reboots → On successful boot: marks B valid, reconnects via ESP-NOW
- **Hub OTA**: Web UI trigger → Operator uploads firmware binary to hub via HTTP → Hub downloads and applies to OTA partition B, reboots

**Persistence (NVS):**

| Device | NVS Keys |
|--------|---------|
| Sensor | Device name, gyroscope calibration offsets, OTA boot partition metadata |
| Hub | Sensor registry (MAC → name map), WiFi AP channel |

---

## 3. Implementation Phases

### 3.1 Phase 1 — Sensor Core (Standalone)

**Scope:**
- BMI160 I2C initialization and chip verification
- Gyroscope calibration at boot; NVS persistence of offsets
- IMU data acquisition loop (gyro + accel)
- Flight detection state machine (Still / Moving / Flying)
- Gyro integration and throw result computation (rotation minor, rotation major, ToF, max accel, validity)
- Battery SoC monitoring
- Serial debug output for all state transitions and results
- No ESP-NOW in this phase; hub connection deferred to Phase 2

**Deliverables:**
- Sensor firmware running on ESP32-C3 with BMI160 attached via I2C
- Serial output showing: boot calibration result, state transitions, throw result fields, battery SoC

**Exit Criteria:**
- Throw start detected within 100 ms of actual throw (manual validation)
- Landing detected within 200 ms of actual impact
- Rotation around minor axis within ± 15° of reference measurement (protractor rig or video analysis)
- Battery SoC (%) logged periodically on serial; value plausible when battery is charged
- State machine transitions correctly for: rest, slow pick-up, throw, hard impact
- No spurious "Flying" state when baton is slowly moved or set down

**Dependencies:** None

---

### 3.2 Phase 2 — Hub Core (Without Web Interface)

**Scope:**
- Sensor firmware updated: ESP-NOW TX (status packets, throw result packets)
- Hub firmware: ESP-NOW RX (star topology, accept all sensors)
- Sensor registry (MAC → name) on hub
- Game statistics computation per sensor
- Hub serial debug output for all received data and statistics
- Sensor name assignment and calibration command relay (via hub serial commands as temporary workaround)

**Deliverables:**
- Updated sensor firmware with ESP-NOW TX
- Hub firmware with ESP-NOW RX and statistics engine
- Hub serial shows: all received throw events, updated statistics per sensor
- ≥ 2 sensors communicating with hub simultaneously (full 6-sensor test at phase end)

**Exit Criteria:**
- Hub receives throw results from all active sensors within 2 s of landing detection
- Game statistics correct: total throws, max/min/avg rotation (minor and major), max/min/avg ToF, max/min/avg max accel
- No data loss when ≥ 2 sensors transmit simultaneously
- Sensor name assigned via hub serial persists in hub NVS after hub reboot

**Dependencies:** Phase 1 complete

---

### 3.3 Phase 3 — Web Interface

**Scope:**
- Hub embedded HTTP server fully operational
- Responsive web interface (SPA) served from hub
- Per-sensor cards: state, last throw data, validity highlight, battery SoC
- Game statistics table per sensor
- Sensor management controls: name assignment, calibration trigger
- Statistics reset button
- OTA update flow (sensor and hub) accessible from web UI

**Deliverables:**
- Web interface accessible at `http://192.168.4.1` from mobile browser on hub AP
- All management functions operational from a smartphone browser
- Successful OTA update verified for at least one sensor and the hub

**Exit Criteria:**
- Interface loads within 3 s on mobile browser connected to hub AP
- Sensor data updates within 1 s of sensor transmission (no page reload required)
- Sensor name change persists across hub reboot
- Statistics reset clears all counters
- OTA flow completes for one sensor and for hub
- Interface usable on iOS Safari and Android Chrome at ≥ 360 px screen width

**Dependencies:** Phase 2 complete

---

## 4. Functional Requirements

### 4.1 Functional Requirements (FR)

#### FR-1: IMU Data Acquisition

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-1.1 | Must | The sensor shall initialize the BMI160 via I2C on boot and verify the chip ID (expected: 0xD1). |
| FR-1.2 | Must | The sensor shall sample the BMI160 gyroscope at ≥ 400 Hz during active operation. |
| FR-1.3 | Must | The sensor shall sample the BMI160 accelerometer at ≥ 100 Hz during active operation. |
| FR-1.4 | Must | The sensor shall configure the BMI160 gyroscope full-scale range to ≥ ±2000 °/s. |
| FR-1.5 | Must | The sensor shall configure the BMI160 accelerometer full-scale range to ≥ ±16 g. |
| FR-1.6 | Must | The sensor shall perform gyroscope zero-rate offset calibration at boot with the sensor held stationary for ≥ 3 s. |
| FR-1.7 | Must | The sensor shall apply calibration offsets to all gyroscope readings in real time. |
| FR-1.8 | Must | The sensor shall persist calibration offsets in NVS and apply them on subsequent boots without re-running calibration. |
| FR-1.9 | Must | The sensor shall support manual calibration re-triggering via a command received from the hub. |

#### FR-2: Flight & Throw Detection

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-2.1 | Must | The sensor shall implement a state machine with states: **Still**, **Moving**, **Flying**. |
| FR-2.2 | Must | Transition Still → Moving shall be triggered when gyroscope magnitude exceeds a configurable threshold (default: 50 °/s) sustained for ≥ 20 ms. |
| FR-2.3 | Must | Transition Moving → Flying (throw detected) shall be triggered when combined gyroscope magnitude and accelerometer deviation from 1 g indicate a ballistic throw pattern. |
| FR-2.4 | Must | Transition Flying → Still (landing detected) shall be triggered by a high-G impact event on the accelerometer exceeding a configurable threshold (default: 4 g peak). |
| FR-2.5 | Should | The sensor shall apply debounce and hysteresis to suppress spurious state transitions caused by slow handling or resting movements. |
| FR-2.6 | Should | The sensor shall buffer the last 100 ms of IMU data continuously (circular pre-buffer) to capture the pre-throw context. |

#### FR-3: Throw Analysis

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-3.1 | Must | Upon landing detection, the sensor shall integrate the gyroscope data over the flight period to compute total rotation around the minor axis (°). |
| FR-3.2 | Must | Upon landing detection, the sensor shall integrate the gyroscope data over the flight period to compute total rotation around the major axis (°). |
| FR-3.3 | Must | The sensor shall compute time-of-flight as the elapsed time between throw start detection and landing detection (ms). |
| FR-3.4 | Must | The sensor shall record the maximum longitudinal acceleration (along the major axis) observed during flight (g). |
| FR-3.5 | Must | The sensor shall determine throw validity: a throw is valid if and only if the rotation around the minor axis during flight is ≥ 360°. |
| FR-3.6 | Must | The sensor shall package the throw result as a structured record and transmit it to the hub via ESP-NOW immediately after analysis completes. |

#### FR-4: Sensor Communication (ESP-NOW)

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-4.1 | Must | The sensor shall communicate with the hub using ESP-NOW in station mode. |
| FR-4.2 | Must | The sensor shall send a periodic STATUS packet to the hub every 1 s containing: sensor MAC, current name, current state, and battery SoC. |
| FR-4.3 | Must | The sensor shall send a THROW_RESULT packet to the hub immediately after throw analysis completes. |
| FR-4.4 | Must | The THROW_RESULT packet shall contain: sensor MAC, throw start uptime (ms), throw end uptime (ms), rotation_minor (°), rotation_major (°), max_longitudinal_accel (g), is_valid (bool), sequence number. |
| FR-4.5 | Should | The sensor shall retransmit the THROW_RESULT packet if no ESP-NOW acknowledgment is received within 500 ms (up to 3 retries). |
| FR-4.6 | Must | The sensor shall receive and process commands from the hub: CMD_SET_NAME, CMD_CALIBRATE, CMD_OTA_START. |

#### FR-5: Sensor Configuration & Identification

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-5.1 | Must | Each sensor shall have a unique user-assigned device name (text string, max 15 characters + null terminator). |
| FR-5.2 | Must | The sensor shall persist its name in NVS across reboots. |
| FR-5.3 | Must | If no name has been assigned, the sensor shall use a MAC-derived default name (e.g., `Baton-A1B2`). |
| FR-5.4 | Must | The sensor name shall be assignable via the hub web interface. |
| FR-5.5 | Must | The sensor shall include its current name in all STATUS packets sent to the hub. |

#### FR-6: Battery Monitoring

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-6.1 | Must | The sensor shall measure the LiIon 1S cell voltage via the ESP32-C3 ADC at intervals ≤ 60 s. |
| FR-6.2 | Must | The sensor shall convert cell voltage to a state-of-charge estimate (%) using a voltage-SoC lookup table for LiIon 1S cells (3.0 V = 0 %, 4.2 V = 100 %). |
| FR-6.3 | Must | The sensor shall include the current SoC in all STATUS packets sent to the hub. |
| FR-6.4 | Should | The sensor shall log a warning via serial when SoC drops below 20 %. |

#### FR-7: Hub — ESP-NOW Reception & Data Storage

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-7.1 | Must | The hub shall act as the central node of an ESP-NOW star topology and receive packets from all registered sensors. |
| FR-7.2 | Must | The hub shall maintain a sensor registry: a list of all sensors that have communicated, keyed by MAC address, storing user-assigned name and last-seen timestamp. |
| FR-7.3 | Must | The hub shall store the most recent STATUS (state, SoC) and last THROW_RESULT for each registered sensor. |
| FR-7.4 | Must | The hub shall compute and update game statistics per sensor upon receiving each THROW_RESULT. |
| FR-7.5 | Must | Game statistics per sensor shall include: total throw count, valid throw count, max/min/avg rotation around minor axis (°), max/min/avg rotation around major axis (°), max/min/avg time-of-flight (ms), max/min/avg maximum longitudinal acceleration (g). |
| FR-7.6 | Must | The hub shall accept a statistics reset command and clear all per-sensor statistics. |
| FR-7.7 | Must | The hub shall send commands to sensors via ESP-NOW: CMD_SET_NAME, CMD_CALIBRATE, CMD_OTA_START. |
| FR-7.8 | Should | The hub shall output all received sensor data and computed statistics via serial debug interface. |

#### FR-8: Hub — WiFi Access Point & HTTP Server

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-8.1 | Must | The hub shall start a WiFi Access Point on boot with SSID `KubbChecker-{HUB_MAC_LAST_4}`. |
| FR-8.2 | Must | The hub shall assign IP addresses to AP clients via DHCP; hub IP shall be 192.168.4.1. |
| FR-8.3 | Must | The hub shall run an embedded HTTP server on port 80 accessible to all WiFi AP clients. |
| FR-8.4 | Must | The hub shall serve the web interface as a self-contained page (HTML/CSS/JS) from its HTTP server. |
| FR-8.5 | Should | The hub HTTP server shall support ≥ 3 simultaneous client connections. |
| FR-8.6 | Should | The hub shall push sensor data updates to web clients at intervals ≤ 1 s (polling or WebSocket). |

#### FR-9: Web Interface — Display

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-9.1 | Must | The web interface shall display one card per registered sensor showing: sensor name, current state (Still / Moving / Flying), last throw rotation around minor axis (°), last throw rotation around major axis (°), last throw time-of-flight (ms), last throw max longitudinal acceleration (g), last throw validity (Valid / Invalid), and battery SoC (%). |
| FR-9.2 | Must | The web interface shall visually distinguish valid throws (green) from invalid throws (red). |
| FR-9.3 | Must | The web interface shall display game statistics per sensor: total throws, valid throw count, max/min/avg rotation minor (°), max/min/avg rotation major (°), max/min/avg ToF (ms), max/min/avg max accel (g). |
| FR-9.4 | Must | The web interface shall display battery SoC for each registered sensor. |
| FR-9.5 | Should | The web interface shall update sensor data without requiring a page reload. |
| FR-9.6 | Should | The web interface shall be responsive and usable on a smartphone screen of ≥ 360 px width. |

#### FR-10: Web Interface — Control

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-10.1 | Must | The web interface shall provide a text input and submit control to assign a name to each sensor. |
| FR-10.2 | Must | The web interface shall provide a button to trigger gyroscope calibration for each sensor. |
| FR-10.3 | Must | The web interface shall provide a button to reset all game statistics. |
| FR-10.4 | Must | The web interface shall provide OTA update controls: upload firmware and trigger update for each sensor and for the hub. |

#### FR-11: OTA Firmware Updates

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-11.1 | Must | All devices (sensors and hub) shall support OTA firmware updates without physical access. |
| FR-11.2 | Must | All devices shall use A/B OTA partition scheme to enable automatic rollback on boot failure. |
| FR-11.3 | Must | All devices shall verify firmware integrity (checksum / ESP-IDF signature) before applying an update. |
| FR-11.4 | Must | All devices shall rollback to the previous firmware partition on boot failure after an OTA update. |
| FR-11.5 | Must | Sensor OTA shall be initiated from the web interface; the sensor shall not require a USB connection. |
| FR-11.6 | Must | During sensor OTA, the sensor shall temporarily connect to the hub's WiFi AP as STA and download firmware via HTTP from the hub (`http://192.168.4.1/firmware/sensor.bin`). |
| FR-11.7 | Should | OTA progress (download %, verification, apply) shall be visible in the serial debug output and in the web interface. |
| FR-11.8 | Should | All devices shall report their running firmware version in the web interface and serial debug output. |

#### FR-12: Serial Debug Output

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-12.1 | Must | The sensor shall output all state transitions, throw events, and throw results via the UART serial interface. |
| FR-12.2 | Must | The sensor shall output IMU calibration offsets and calibration status at boot via serial. |
| FR-12.3 | Must | The sensor shall output battery SoC readings periodically via serial. |
| FR-12.4 | Must | The hub shall output all received sensor data and computed statistics via serial. |
| FR-12.5 | Should | All serial log messages shall include a timestamp and a component tag (e.g., `[IMU]`, `[ESPNOW]`, `[FLIGHT]`). |

---

### 4.2 Non-Functional Requirements (NFR)

| ID | Priority | Requirement |
|----|----------|-------------|
| NFR-1.1 | Must | Rotation measurement around the minor axis shall have absolute error ≤ 15° for throws with rotation between 90° and 1080° at angular rates up to 1800 °/s. |
| NFR-1.2 | Must | Throw start detection shall occur within 100 ms of the baton leaving the thrower's hand. |
| NFR-1.3 | Must | Landing detection shall occur within 200 ms of the physical impact event. |
| NFR-2.1 | Must | The system shall support ≥ 6 simultaneous sensor nodes communicating with a single hub without data loss. |
| NFR-2.2 | Should | ESP-NOW throw result delivery from sensor to hub shall complete within 2 s of landing detection. |
| NFR-2.3 | Should | Web interface sensor data shall refresh within 1 s of the hub receiving new data. |
| NFR-3.1 | Should | The sensor shall operate for ≥ 3 hours continuously on a single battery charge under normal game usage. |
| NFR-3.2 | Should | The sensor shall enter reduced-power idle mode in the Still state to extend battery life. |
| NFR-4.1 | Should | The hub HTTP server shall support ≥ 3 simultaneous browser clients without perceptible degradation. |
| NFR-5.1 | Must | The sensor shall independently detect and record a second throw occurring within 5 s of the previous throw landing. |
| NFR-5.2 | Should | The hub shall not lose throw records when two sensors transmit simultaneously. |

---

### 4.3 Constraints

| ID | Constraint |
|----|-----------|
| CON-1 | Sensor firmware shall target ESP-IDF (or Arduino framework on ESP-IDF) for the ESP32-C3. |
| CON-2 | Hub firmware shall target ESP-IDF (or Arduino framework on ESP-IDF) for the ESP32. |
| CON-3 | Sensor PCB and LiIon 1S battery shall fit within a baton of approximately 300 mm × 40 mm diameter. |
| CON-4 | The hub shall not require internet connectivity for any operational function. |
| CON-5 | The ESP-NOW protocol limits unicast payload to 250 bytes per frame. All packet structures shall remain within this limit. |
| CON-6 | The BMI160 I2C address is 0x68 (SDO = GND) unless hardware requires 0x69 (SDO = VDD). |
| CON-7 | OTA partition scheme requires ≥ 2 MB flash on each device; ESP32-C3 and ESP32 modules shall be selected accordingly. |
| CON-8 | ESP-NOW and WiFi AP on the hub must operate on the same 2.4 GHz channel. |

---

## 5. Risks, Assumptions & Dependencies

### 5.1 Risk Register

| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|-----------|--------|-----------|
| R-1 | Gyro integration drift causes rotation error > 15° | Medium | High | Sample at ≥ 400 Hz; apply calibration offsets; limit integration to confirmed flight window only |
| R-2 | False throw detection from rough handling or setting down baton | Medium | Medium | Implement debounce and hysteresis; tune thresholds during field testing |
| R-3 | ESP-NOW range insufficient across a full Kubb field (~10 m) | Low | High | Test range in open field early; 2.4 GHz ESP-NOW with +20 dBm TX power exceeds 10 m typical range |
| R-4 | Concurrent ESP-NOW and WiFi AP on hub causes packet loss or interference | Low | High | Use identical channel for both; validate during Phase 2 multi-sensor test |
| R-5 | Sensor hardware (PCB + LiIon) does not fit inside baton | Medium | High | Validate physical dimensions before PCB layout; custom PCB may be required |
| R-6 | Battery SoC voltage estimate is inaccurate under load | Medium | Low | Acceptable for game use; note in documentation; improve with coulomb counting if needed |
| R-7 | Hub has no absolute time reference (no NTP, no RTC) | High | Low | Hub uses uptime-relative timestamps for all throw events; user may set wall-clock time via web UI if needed |
| R-8 | Sensor is non-operational during OTA download | Medium | Medium | Document the OTA window; do not trigger during active game play |

### 5.2 Assumptions

| ID | Assumption |
|----|-----------|
| A-1 | The BMI160 is rigidly fixed inside the baton with one gyroscope axis aligned to the baton's minor axis. The axis mapping (BMI160 axis ↔ baton minor / major) is defined at compile time. |
| A-2 | The hub's ESP32 supports simultaneous ESP-NOW and WiFi AP operation on the same 2.4 GHz channel (standard ESP-IDF capability). |
| A-3 | Throw timestamps are hub-uptime-relative at the time of ESP-NOW packet receipt, not absolute wall-clock time. (assumed) |
| A-4 | The hub is powered by an external supply and has no battery constraints. (assumed) |
| A-5 | The hub WiFi AP operates with no password (open authentication) for ease of use at the game site. (assumed) |
| A-6 | Game statistics are stored in hub RAM and are lost on hub power-cycle; NVS persistence of statistics is deferred. (assumed) |
| A-7 | Sensor firmware and hub firmware are separate build projects sharing a common protocol header. |
| A-8 | Maximum throw angular velocity is ≤ 1800 °/s; the ±2000 °/s gyro range provides sufficient headroom. (assumed) |
| A-9 | A BMI160 I2C driver is available (Bosch BSX or community library); if not, a minimal register-level driver will be written. |

### 5.3 External Dependencies

| Dependency | Owner | Notes |
|-----------|-------|-------|
| BMI160 I2C driver / library | Third-party or custom | Verify availability; fallback to register-level driver using `Documentation/bst-bmi160-ds000.pdf` |
| ESP-IDF ESP-NOW API | Espressif | Use ESP-IDF LTS; API is stable |
| ESP32-C3 / ESP32 toolchain | Espressif | ESP-IDF v5.x recommended |
| LiIon 1S charging circuit | Hardware design | Charging is out of firmware scope; battery monitoring reads cell voltage only |

---

## 6. Interface Specifications

### 6.1 External Interfaces

#### 6.1.1 WiFi Access Point

| Attribute | Value |
|-----------|-------|
| SSID | `KubbChecker-{HUB_MAC_LAST_4}` |
| Authentication | Open (no password) (assumed) |
| Channel | 1 (default; must match ESP-NOW channel) |
| Hub IP (gateway) | 192.168.4.1 |
| DHCP range | 192.168.4.2 – 192.168.4.20 |

#### 6.1.2 HTTP REST API

All endpoints served by the hub at `http://192.168.4.1/`.

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|-------------|----------|
| GET | `/` | Serve web interface SPA | — | HTML |
| GET | `/api/sensors` | List all sensors with current state and last throw | — | JSON array |
| GET | `/api/statistics` | Game statistics per sensor | — | JSON object |
| POST | `/api/sensor/{mac}/name` | Assign name to sensor | `{"name":"Baton-A"}` | 200 OK |
| POST | `/api/sensor/{mac}/calibrate` | Trigger gyro calibration | — | 200 OK |
| POST | `/api/statistics/reset` | Reset all game statistics | — | 200 OK |
| GET | `/api/ota/status` | Firmware version for hub and all sensors | — | JSON |
| POST | `/api/ota/upload/sensor` | Upload sensor firmware binary to hub | Binary (multipart) | 200 OK |
| POST | `/api/ota/upload/hub` | Upload hub firmware binary | Binary (multipart) | 200 OK |
| POST | `/api/ota/sensor/{mac}` | Trigger OTA update for a specific sensor | — | 200 OK |
| POST | `/api/ota/hub` | Trigger OTA self-update for hub | — | 200 OK |

Sensor OTA firmware is served from:
```
GET http://192.168.4.1/firmware/sensor.bin
```

---

### 6.2 Internal Interfaces

#### 6.2.1 I2C: ESP32-C3 ↔ BMI160

| Attribute | Value |
|-----------|-------|
| Protocol | I2C |
| Frequency | 400 kHz (Fast Mode) |
| BMI160 address | 0x68 (SDO = GND) |
| Data read | Gyroscope (X, Y, Z), Accelerometer (X, Y, Z) |
| Mode | Polling (interrupt-driven as optional optimization) |

#### 6.2.2 ESP-NOW: Sensor ↔ Hub

| Attribute | Value |
|-----------|-------|
| Protocol | ESP-NOW (Espressif, IEEE 802.11-based) |
| Topology | Star; hub MAC pre-configured in sensor firmware |
| Max payload | 250 bytes |
| Channel | Must match hub WiFi AP channel |
| Encryption | Disabled (assumed) |
| Delivery mode | Unicast with hardware ACK |

---

### 6.3 Data Models / Schemas

#### 6.3.1 ESP-NOW Packet Structures

All packets share a 1-byte `msg_type` header. All multi-byte integers are little-endian.

**Message Type Enum:**

| Value | Name | Direction |
|-------|------|-----------|
| 0x01 | STATUS | Sensor → Hub |
| 0x02 | THROW_RESULT | Sensor → Hub |
| 0x10 | CMD_SET_NAME | Hub → Sensor |
| 0x11 | CMD_CALIBRATE | Hub → Sensor |
| 0x12 | CMD_OTA_START | Hub → Sensor |

**STATUS packet (29 bytes):**

```c
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;       // 0x01
    uint8_t  mac[6];         // sensor MAC address
    uint8_t  state;          // 0=Still, 1=Moving, 2=Flying
    uint8_t  battery_soc;   // 0–100 %
    uint32_t uptime_ms;     // sensor uptime at send time
    char     name[15];       // user-assigned name, null-terminated
    uint8_t  _pad;           // reserved
} status_pkt_t;              // total: 29 bytes
```

**THROW_RESULT packet (29 bytes):**

```c
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;           // 0x02
    uint8_t  mac[6];             // sensor MAC address
    uint32_t throw_start_ms;    // sensor uptime at throw start
    uint32_t throw_end_ms;      // sensor uptime at landing
    float    rotation_minor;    // integrated rotation, minor axis (degrees)
    float    rotation_major;    // integrated rotation, major axis (degrees)
    float    max_accel_long;    // max longitudinal acceleration (g)
    uint8_t  is_valid;          // 1 if rotation_minor >= 360, else 0
    uint8_t  seq_num;           // rolling sequence number (for dedup/retry)
} throw_result_pkt_t;           // total: 29 bytes
```

**CMD_SET_NAME packet (17 bytes):**

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;   // 0x10
    char    name[15];   // new name, null-terminated
    uint8_t _pad;
} cmd_set_name_t;
```

**CMD_CALIBRATE packet (2 bytes):**

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;   // 0x11
    uint8_t _pad;
} cmd_calibrate_t;
```

**CMD_OTA_START packet (6 bytes):**

```c
typedef struct __attribute__((packed)) {
    uint8_t msg_type;    // 0x12
    uint8_t hub_ip[4];  // hub IP for HTTP download (e.g., {192,168,4,1})
    uint8_t _pad;
} cmd_ota_start_t;
```

#### 6.3.2 HTTP JSON Schemas

**GET /api/sensors:**

```json
{
  "sensors": [
    {
      "mac": "AA:BB:CC:DD:EE:FF",
      "name": "Baton-A",
      "state": "Still",
      "battery_soc": 87,
      "last_seen_ms": 1500,
      "firmware_version": "1.0.0",
      "last_throw": {
        "timestamp_ms": 1234567,
        "rotation_minor_deg": 412.3,
        "rotation_major_deg": 25.7,
        "time_of_flight_ms": 850,
        "max_accel_g": 15.2,
        "is_valid": true
      }
    }
  ]
}
```

**GET /api/statistics:**

```json
{
  "statistics": [
    {
      "mac": "AA:BB:CC:DD:EE:FF",
      "name": "Baton-A",
      "total_throws": 12,
      "valid_throws": 9,
      "rotation_minor_deg": { "min": 234.1, "max": 856.3, "avg": 421.7 },
      "rotation_major_deg": { "min": 5.2,   "max": 89.3,  "avg": 31.1 },
      "time_of_flight_ms":  { "min": 620,   "max": 1250,  "avg": 890  },
      "max_accel_g":        { "min": 8.1,   "max": 22.4,  "avg": 14.2 }
    }
  ]
}
```

---

### 6.4 Commands / Opcodes

ESP-NOW command packets are defined in Section 6.3.1. HTTP control endpoints are defined in Section 6.1.2.

---

## 7. Operational Procedures

### 7.1 First-Time Deployment

#### 7.1.1 Hub Setup

1. Flash hub firmware via USB serial (initial flash only).
2. Power on hub.
3. Verify serial output: "Hub ready", SSID `KubbChecker-XXXX` logged.
4. Connect mobile device to the hub's AP.
5. Open browser at `http://192.168.4.1` and verify the web interface loads.

#### 7.1.2 Sensor Setup (per sensor)

1. Flash sensor firmware via USB serial (initial flash only).
2. Place sensor (or baton) on a flat, stationary surface.
3. Power on sensor.
4. Wait ≥ 3 s for gyroscope calibration to complete (serial logs offsets).
5. Sensor registers with hub automatically on first ESP-NOW contact.
6. In the hub web interface, locate the sensor by its MAC-derived default name.
7. Assign a meaningful name (e.g., "Baton-1") via the web interface.

#### 7.1.3 Full Deployment

Repeat sensor setup for all ≥ 6 sensors. Verify all sensors appear in the hub web interface before starting the game.

---

### 7.2 Normal Game Operation

1. Power on hub and all sensors at the game location.
2. Sensors auto-calibrate at boot (hold or lay flat during power-on).
3. Connect referee's mobile device to `KubbChecker-XXXX` AP.
4. Open web interface; confirm all sensors are visible and show "Still".
5. During game: after each throw, the web interface displays rotation, validity (green / red), and updated statistics.
6. At end of game: press **Reset Statistics** for the next game session.

---

### 7.3 Manual Gyroscope Calibration

1. Place target baton on a flat, stationary surface.
2. In the web interface, press **Calibrate** on the sensor's card.
3. Hub sends CMD_CALIBRATE via ESP-NOW.
4. Sensor runs calibration for ≥ 3 s; updated offsets stored in NVS.
5. Serial debug confirms new calibration offsets.

---

### 7.4 Sensor Renaming

1. In the web interface, locate the sensor card.
2. Enter a new name (max 15 characters) in the name field.
3. Press **Set Name**.
4. Hub sends CMD_SET_NAME via ESP-NOW.
5. Sensor saves new name to NVS.
6. Sensor includes new name in the next STATUS packet; web interface updates within ≤ 2 s.

---

### 7.5 OTA Firmware Update — Sensor

1. Build the new sensor firmware binary (`sensor.bin`).
2. In the web interface: **Settings → OTA → Upload Sensor Firmware**; select `sensor.bin`.
3. Locate the target sensor and press **Update Firmware**.
4. Hub sends CMD_OTA_START via ESP-NOW.
5. Sensor suspends normal operation; connects to hub AP as WiFi STA.
6. Sensor downloads `http://192.168.4.1/firmware/sensor.bin`.
7. Sensor verifies checksum, writes to OTA partition B, and reboots.
8. Sensor reconnects to hub via ESP-NOW; web interface shows updated firmware version.

> **Note:** The sensor does not detect throws during OTA. Complete active game rounds before updating.

---

### 7.6 OTA Firmware Update — Hub

1. Build the new hub firmware binary (`hub.bin`).
2. In the web interface: **Settings → OTA → Upload Hub Firmware**; select `hub.bin`.
3. Press **Update Hub Firmware**.
4. Hub writes firmware to OTA partition B and reboots (≈ 30–60 s downtime).
5. Web clients reconnect automatically after hub reboots; verify updated version in UI.

---

### 7.7 Statistics Reset

1. Press **Reset Statistics** in the web interface.
2. All game statistics (throw counts, min/max/avg values) are cleared in hub RAM.
3. Individual sensor last-throw records are preserved; only aggregated statistics are reset.

---

### 7.8 Recovery Procedures

**Sensor missing from web interface (no status for > 10 s):**
1. Check sensor serial for boot errors or calibration hang.
2. Power-cycle the sensor.
3. If calibration fails, ensure sensor is held stationary at power-on.

**Hub web interface unreachable:**
1. Verify mobile is connected to `KubbChecker-XXXX` (not to home WiFi).
2. Navigate explicitly to `http://192.168.4.1`.
3. If still unreachable, check hub serial for HTTP server errors.
4. Power-cycle hub.

**Sensor unresponsive after OTA (boot loop):**
1. Power-cycle sensor; A/B rollback should restore the previous firmware automatically.
2. Monitor serial for "rolling back to previous partition".
3. If rollback does not succeed: connect via USB serial and flash factory firmware.

**Hub unresponsive after OTA:**
1. Power-cycle hub; A/B rollback triggers automatically on repeated boot failure.
2. If rollback does not succeed: connect via USB serial and flash factory firmware.

---

## 8. Verification & Validation

### 8.1 Phase 1 Verification — Sensor Core (Standalone)

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-1.1 | BMI160 Init | Power on sensor; monitor serial | "BMI160 OK, chip_id=0xD1" logged; no I2C error |
| TC-1.2 | Gyro Calibration | Power on with sensor flat and still; wait 5 s | Calibration offsets logged; all offsets < 5 °/s |
| TC-1.3 | State: Still → Moving | Slowly pick up baton | Serial shows state transition to "Moving" |
| TC-1.4 | State: Moving → Flying → Still | Perform a throw; baton hits wall | Serial shows "Flying", then "Still" with throw result |
| TC-1.5 | Rotation Measurement (minor) | Use motorized rig for exactly 360° minor rotation | Serial reports 360° ± 15° |
| TC-1.6 | Rotation Measurement (major) | Rotate 180° around major axis manually | Serial reports 180° ± 15° |
| TC-1.7 | Time-of-Flight | Throw with simultaneous stopwatch | Serial ToF within ± 100 ms of stopwatch |
| TC-1.8 | Max Acceleration | Throw onto hard surface | Max accel logged; value between 5 g and 40 g |
| TC-1.9 | Validity: Valid | Perform throw with confirmed ≥ 360° rotation | Serial shows `is_valid=true` |
| TC-1.10 | Validity: Invalid | Perform throw with confirmed < 360° rotation | Serial shows `is_valid=false` |
| TC-1.11 | Battery SoC | Monitor serial for 5 min with charged battery | SoC value (%) logged; > 50% when battery fully charged |
| TC-1.12 | Spurious throw rejection | Slowly pick up and set down baton | No "Flying" state triggered |
| TC-1.13 | Rapid successive throws | Two throws within 5 s | Both throws independently recorded in serial |

---

### 8.2 Phase 2 Verification — Hub Core

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-2.1 | Hub Boot | Power on hub; monitor serial | "Hub ready", ESP-NOW and WiFi AP initialized logged |
| TC-2.2 | Sensor Registration | Power on sensor near hub | Hub serial logs sensor MAC as registered |
| TC-2.3 | STATUS Reception | Monitor hub serial for 30 s | Hub logs sensor STATUS packets at ~1 s interval |
| TC-2.4 | THROW_RESULT Reception | Perform throw with sensor | Hub serial shows throw result within 2 s of landing |
| TC-2.5 | Validity Echo | Perform valid and invalid throw | Hub correctly mirrors Valid / Invalid from sensor |
| TC-2.6 | Multi-Sensor (6 units) | Power on all 6 sensors | Hub registers all 6; no dropped STATUS packets over 2 min |
| TC-2.7 | Statistics Computation | Perform 5 throws with same sensor | Hub serial shows correct min/max/avg after each throw |
| TC-2.8 | Statistics Reset | Issue reset via hub serial | All statistics cleared; confirmed in subsequent serial output |
| TC-2.9 | Battery SoC Relay | Monitor hub serial | SoC value from sensor visible on hub |
| TC-2.10 | Sensor Rename | Send name command via hub serial | Hub stores name; sensor includes new name in next STATUS |
| TC-2.11 | Calibration Command | Send calibration command via hub serial | Sensor serial confirms calibration run |
| TC-2.12 | Concurrent Transmission | Two sensors throw within 500 ms of each other | Both throw results received by hub within 3 s; no corruption |

---

### 8.3 Phase 3 Verification — Web Interface

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-3.1 | Hub AP Visible | Power on hub | `KubbChecker-XXXX` SSID visible on mobile WiFi scan |
| TC-3.2 | Web Interface Load | Connect mobile to AP; navigate to 192.168.4.1 | Interface loads within 3 s |
| TC-3.3 | Sensor Card Appears | Power on sensor | Sensor card visible in UI within 2 s of first STATUS |
| TC-3.4 | State Display Update | Move sensor | UI shows state change within 1 s; no page reload |
| TC-3.5 | Throw Display | Perform throw | UI shows rotation, ToF, accel, validity within 2 s |
| TC-3.6 | Validity Highlight | Perform valid and invalid throw | Valid: green highlight; Invalid: red highlight |
| TC-3.7 | Statistics Update | Perform 3 throws | Statistics (avg, min, max) update correctly in UI |
| TC-3.8 | Statistics Reset | Click Reset button | All statistics cleared; confirmed immediately in UI |
| TC-3.9 | Sensor Rename | Enter new name; click Set Name | Name updates in UI within 2 s; persists after hub reboot |
| TC-3.10 | Battery Display | Check SoC in UI | SoC percentage visible and plausible for each sensor |
| TC-3.11 | Calibration via UI | Click Calibrate button | Sensor serial confirms calibration run within 5 s |
| TC-3.12 | Multi-Client | Connect 3 phones to AP simultaneously | All 3 show identical data; no crash or slowdown |
| TC-3.13 | Responsive Layout | Load UI on 360 px wide phone | All elements visible without horizontal scroll |

---

### 8.4 OTA Verification

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-OTA-100 | Sensor OTA Success | Upload firmware V2; trigger sensor OTA from UI | Sensor reboots with V2 firmware; reconnects to hub; UI shows new version |
| TC-OTA-101 | Sensor OTA Rollback | Flash intentionally broken sensor firmware via OTA | Sensor rolls back to V1 automatically; normal operation resumes |
| TC-OTA-102 | Hub OTA Success | Upload firmware V2; trigger hub OTA | Hub reboots with V2; web UI functional; version updated |
| TC-OTA-103 | Version Reporting | Check firmware version in UI after OTA | Correct version string displayed for hub and all sensors |
| EC-OTA-200 | Network Loss During OTA | Drop WiFi connection at 50% download | No brick; sensor reverts to normal; OTA retry succeeds |
| EC-OTA-201 | Power Loss During OTA Write | Cut power mid-flash | Previous firmware boots on power restore (A/B scheme) |
| EC-OTA-202 | Invalid Firmware Rejection | Upload random binary; trigger sensor OTA | Rejected (checksum fail); original firmware intact |
| EC-OTA-203 | OTA During Active Use | Trigger OTA while hub is receiving throw events from other sensors | Other sensors continue operating; hub serves OTA file; no crash |

---

### 8.5 Serial Logging Verification

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-LOG-100 | Serial Boot Log | Power on sensor; monitor serial | Firmware version, chip ID, calibration result logged at boot |
| TC-LOG-103 | State Transition Logging | Cycle sensor through Still → Moving → Flying → Still | All transitions logged with timestamp and component tag |
| EC-LOG-203 | Crash Log Capture | Trigger crash (e.g., stack overflow test) | Backtrace and reset reason visible on serial |

---

### 8.6 Acceptance Tests

| Test ID | Scenario | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| AT-1 | Full Game Session | Deploy 6 sensors + hub; play a complete game (≥ 20 throws) | All throws recorded; statistics correct; no data loss |
| AT-2 | Valid Throw Confirmation | Throw with confirmed ≥ 360° rotation (video reference at 240 fps) | System reports Valid; measured rotation within ± 15° of video analysis |
| AT-3 | Invalid Throw Confirmation | Throw with confirmed < 360° rotation (held back deliberately) | System reports Invalid |
| AT-4 | Long Session (3 hours) | Run system for 3 hours with throw activity every few minutes | No sensor battery failure; no hub crash; all statistics intact |
| AT-5 | OTA During Game Pause | Trigger OTA update for one sensor between game rounds | OTA completes; sensor reconnects; game statistics preserved |
| AT-6 | Multi-Client Real-Time | Referee + 3 spectators connected to hub AP during active game | All 4 clients see identical real-time throw results |

---

### 8.7 Traceability Matrix

| Requirement | Priority | Test Case(s) | Status |
|-------------|----------|-------------|--------|
| FR-1.1 | Must | TC-1.1 | Covered |
| FR-1.2 | Must | TC-1.5, TC-1.6 | Covered |
| FR-1.3 | Must | TC-1.5, TC-1.8 | Covered |
| FR-1.4 | Must | TC-1.5, TC-1.6 | Covered |
| FR-1.5 | Must | TC-1.8 | Covered |
| FR-1.6 | Must | TC-1.2, TC-1.5 | Covered |
| FR-1.7 | Must | TC-1.2, TC-1.5 | Covered |
| FR-1.8 | Must | TC-1.2 | Covered |
| FR-1.9 | Must | TC-3.11 | Covered |
| FR-2.1 | Must | TC-1.3, TC-1.4 | Covered |
| FR-2.2 | Must | TC-1.3 | Covered |
| FR-2.3 | Must | TC-1.4, AT-2 | Covered |
| FR-2.4 | Must | TC-1.4 | Covered |
| FR-2.5 | Should | TC-1.12 | Covered |
| FR-2.6 | Should | — | GAP |
| FR-3.1 | Must | TC-1.5, AT-2 | Covered |
| FR-3.2 | Must | TC-1.6 | Covered |
| FR-3.3 | Must | TC-1.7 | Covered |
| FR-3.4 | Must | TC-1.8 | Covered |
| FR-3.5 | Must | TC-1.9, TC-1.10, AT-3 | Covered |
| FR-3.6 | Must | TC-2.4 | Covered |
| FR-4.1 | Must | TC-2.2 | Covered |
| FR-4.2 | Must | TC-2.3 | Covered |
| FR-4.3 | Must | TC-2.4 | Covered |
| FR-4.4 | Must | TC-2.4, TC-2.5 | Covered |
| FR-4.5 | Should | TC-2.12 | Covered |
| FR-4.6 | Must | TC-2.10, TC-2.11 | Covered |
| FR-5.1 | Must | TC-2.10, TC-3.9 | Covered |
| FR-5.2 | Must | TC-3.9 | Covered |
| FR-5.3 | Must | TC-2.2 | Covered |
| FR-5.4 | Must | TC-3.9 | Covered |
| FR-5.5 | Must | TC-2.3 | Covered |
| FR-6.1 | Must | TC-1.11 | Covered |
| FR-6.2 | Must | TC-1.11 | Covered |
| FR-6.3 | Must | TC-2.9 | Covered |
| FR-6.4 | Should | — | GAP |
| FR-7.1 | Must | TC-2.2, TC-2.6 | Covered |
| FR-7.2 | Must | TC-2.2, TC-3.9 | Covered |
| FR-7.3 | Must | TC-2.4, TC-3.5 | Covered |
| FR-7.4 | Must | TC-2.7 | Covered |
| FR-7.5 | Must | TC-2.7, TC-3.7 | Covered |
| FR-7.6 | Must | TC-2.8, TC-3.8 | Covered |
| FR-7.7 | Must | TC-2.10, TC-2.11 | Covered |
| FR-7.8 | Should | TC-2.4 | Covered |
| FR-8.1 | Must | TC-3.1 | Covered |
| FR-8.2 | Must | TC-3.1 | Covered |
| FR-8.3 | Must | TC-3.2 | Covered |
| FR-8.4 | Must | TC-3.2 | Covered |
| FR-8.5 | Should | TC-3.12 | Covered |
| FR-8.6 | Should | TC-3.4, TC-3.5 | Covered |
| FR-9.1 | Must | TC-3.3, TC-3.5 | Covered |
| FR-9.2 | Must | TC-3.6 | Covered |
| FR-9.3 | Must | TC-3.7 | Covered |
| FR-9.4 | Must | TC-3.10 | Covered |
| FR-9.5 | Should | TC-3.4, TC-3.5 | Covered |
| FR-9.6 | Should | TC-3.13 | Covered |
| FR-10.1 | Must | TC-3.9 | Covered |
| FR-10.2 | Must | TC-3.11 | Covered |
| FR-10.3 | Must | TC-3.8 | Covered |
| FR-10.4 | Must | TC-OTA-100, TC-OTA-102 | Covered |
| FR-11.1 | Must | TC-OTA-100, TC-OTA-102 | Covered |
| FR-11.2 | Must | TC-OTA-101, EC-OTA-201 | Covered |
| FR-11.3 | Must | EC-OTA-202 | Covered |
| FR-11.4 | Must | TC-OTA-101, EC-OTA-201 | Covered |
| FR-11.5 | Must | TC-OTA-100 | Covered |
| FR-11.6 | Must | TC-OTA-100 | Covered |
| FR-11.7 | Should | TC-OTA-103 | Covered |
| FR-11.8 | Should | TC-OTA-103 | Covered |
| FR-12.1 | Must | TC-1.3, TC-1.4, TC-LOG-103 | Covered |
| FR-12.2 | Must | TC-1.2, TC-LOG-100 | Covered |
| FR-12.3 | Must | TC-1.11 | Covered |
| FR-12.4 | Must | TC-2.4, TC-2.7 | Covered |
| FR-12.5 | Should | TC-LOG-100 | Covered |
| NFR-1.1 | Must | TC-1.5, TC-1.6, AT-2 | Covered |
| NFR-1.2 | Must | TC-1.4, AT-2 | Covered |
| NFR-1.3 | Must | TC-1.4, AT-2 | Covered |
| NFR-2.1 | Must | TC-2.6, AT-1 | Covered |
| NFR-2.2 | Should | TC-2.4 | Covered |
| NFR-2.3 | Should | TC-3.5 | Covered |
| NFR-3.1 | Should | AT-4 | Covered |
| NFR-3.2 | Should | AT-4 | Covered |
| NFR-4.1 | Should | TC-3.12 | Covered |
| NFR-5.1 | Must | TC-1.13 | Covered |
| NFR-5.2 | Should | TC-2.12 | Covered |

**GAP Summary:**

| Requirement | Gap Reason |
|-------------|-----------|
| FR-2.6 | Pre-buffer functional test requires instrumented firmware or logic analyzer; deferred to hardware bring-up |
| FR-6.4 | Low-battery warning observable via serial; no automated test defined — add if test harness allows battery discharge simulation |

---

## 9. Troubleshooting Guide

| Symptom | Likely Cause | Diagnostic Steps | Corrective Action |
|---------|-------------|-----------------|-------------------|
| Sensor not visible on hub web UI | ESP-NOW not reaching hub; sensor not booted | Check sensor serial for boot log; verify hub MAC is correct in sensor firmware | Power-cycle sensor; confirm hub was started first |
| Sensor always shows "Still" when baton is moving | Gyro threshold too high or calibration failed | Check serial: are gyro readings near zero? Is "calibration complete" logged? | Re-run calibration; hold sensor flat and still at power-on |
| All throws report "Invalid" despite visible rotation | BMI160 axis mapping incorrect; minor/major axis swapped | Check serial: compare `rotation_minor` vs `rotation_major` values | Swap the axis mapping constant in firmware; reflash |
| Rotation value wildly wrong (e.g., 2000°) | Integration drift; sensor moved during calibration | Check calibration offsets on serial; repeat calibration on still surface | Re-calibrate; if persistent, verify I2C signal integrity |
| Hub web UI not loading | Mobile on wrong network or wrong URL | Verify mobile is on `KubbChecker-XXXX`; try `http://192.168.4.1` explicitly | Disconnect from home WiFi; reconnect to hub AP |
| Web UI data is stale (no updates) | Polling/WebSocket disconnected | Check browser console for network errors | Reload page |
| Sensor OTA fails; sensor silent after reboot | Bad firmware or interrupted flash | Monitor sensor serial for rollback log | Power-cycle; A/B rollback should restore V1; if not, flash via USB |
| Hub crashes / reboots after hub OTA | Bad hub firmware | Hub auto-reboots; A/B rollback activates | Power-cycle hub; if rollback fails, flash via USB |
| Battery SoC always 0% | ADC pin misconfigured or voltage divider wrong | Check serial for raw ADC reading | Verify hardware voltage divider ratio; adjust ADC scaling constant |
| Two sensors have the same display name | Both assigned identical name string | Identify by MAC address in UI | Rename one sensor with a unique name |
| Spurious "Flying" events during pre-game setup | Detection thresholds too sensitive | Check serial for false "Flying" during slow movement | Increase `MOVING_TO_FLYING` gyro threshold and/or minimum flight duration |
| ESP-NOW packet loss at far end of field | RF range or channel interference | Check hub serial for missing STATUS packets from specific sensor | Move hub closer to playing field center; change WiFi channel |

---

## 10. Appendix

### 10.1 Configuration Defaults

| Parameter | Default | Where |
|-----------|---------|-------|
| Gyro sampling rate | 400 Hz | Sensor firmware |
| Accel sampling rate | 100 Hz | Sensor firmware |
| Gyro full-scale range | ±2000 °/s | Sensor firmware |
| Accel full-scale range | ±16 g | Sensor firmware |
| Still → Moving threshold | 50 °/s gyro magnitude | Sensor firmware |
| Flying → Still impact threshold | 4 g peak accel | Sensor firmware |
| Calibration hold duration | 3 s | Sensor firmware |
| STATUS packet interval | 1000 ms | Sensor firmware |
| THROW_RESULT retry timeout | 500 ms | Sensor firmware |
| THROW_RESULT max retries | 3 | Sensor firmware |
| LiIon SoC 0 % voltage | 3.00 V | Sensor firmware |
| LiIon SoC 100 % voltage | 4.20 V | Sensor firmware |
| BMI160 I2C address | 0x68 | Sensor firmware |
| I2C clock frequency | 400 kHz | Sensor firmware |
| Hub WiFi AP SSID | `KubbChecker-{MAC4}` | Hub firmware |
| Hub WiFi AP channel | 1 | Hub firmware |
| Hub IP address | 192.168.4.1 | Hub firmware |
| DHCP pool | 192.168.4.2 – 192.168.4.20 | Hub firmware |
| Serial baud rate | 115200 | Both |

---

### 10.2 BMI160 Key Registers

| Register | Address | Description |
|----------|---------|-------------|
| CHIP_ID | 0x00 | Shall read 0xD1 to confirm device present |
| GYR_X_L/H | 0x0C/0x0D | Gyroscope X raw (16-bit, signed) |
| GYR_Y_L/H | 0x0E/0x0F | Gyroscope Y raw |
| GYR_Z_L/H | 0x10/0x11 | Gyroscope Z raw |
| ACC_X_L/H | 0x12/0x13 | Accelerometer X raw (16-bit, signed) |
| ACC_Y_L/H | 0x14/0x15 | Accelerometer Y raw |
| ACC_Z_L/H | 0x16/0x17 | Accelerometer Z raw |
| ACC_CONF | 0x40 | Accel ODR and bandwidth |
| ACC_RANGE | 0x41 | Accel full-scale range (0x0C = ±16 g) |
| GYR_CONF | 0x42 | Gyro ODR and bandwidth |
| GYR_RANGE | 0x43 | Gyro full-scale range (0x00 = ±2000 °/s) |
| CMD | 0x7E | Command register (0xB6 = soft reset) |

Full register map: `Documentation/bst-bmi160-ds000.pdf`.

---

### 10.3 LiIon 1S SoC Voltage Lookup Table

| Cell Voltage (V) | SoC (%) |
|------------------|---------|
| ≥ 4.20 | 100 |
| 4.10 | 90 |
| 3.95 | 75 |
| 3.80 | 60 |
| 3.70 | 50 |
| 3.60 | 35 |
| 3.50 | 20 |
| 3.30 | 10 |
| ≤ 3.00 | 0 |

Interpolate linearly between table entries.

---

### 10.4 ESP-NOW / WiFi AP Channel Alignment

ESP-NOW frames are sent on the current primary WiFi channel. The hub WiFi AP channel must match. Steps to ensure alignment:

1. Set hub WiFi AP to channel 1 (or any chosen channel).
2. Sensors automatically use the channel of the hub's AP beacon.
3. If an external AP on channel 1 causes interference, change hub to channel 6 or 11 and update the compile-time default.

---

### 10.5 Throw Detection State Machine

```
         ┌──────────────────────────────────────────────┐
         │ (initial state / after landing)               │
         ▼                                               │
    ┌─────────┐    gyro mag > 50°/s                     │
    │  STILL  │ ─────────────────────► ┌─────────┐      │
    └─────────┘    sustained ≥ 20 ms   │  MOVING │      │
                                        └─────────┘      │
                                             │           │
                              throw pattern  │           │
                              detected       ▼           │
                                        ┌─────────┐     │
                                        │  FLYING │     │
                                        └─────────┘     │
                                             │           │
                               impact ≥ 4 g │           │
                                             ▼           │
                                      [Compute result]   │
                                      [TX THROW_RESULT]  │
                                             └───────────┘
```

---

### 10.6 Sensor OTA Sequence

```
[Web UI: click "Update Firmware" for sensor MAC]
         │
         ▼
[Hub: POST /api/ota/sensor/{mac}]
         │
         ▼
[Hub: send CMD_OTA_START via ESP-NOW → sensor]
         │
         ▼
[Sensor: suspend ESP-NOW, start WiFi STA]
         │
         ▼
[Sensor: connect to KubbChecker-XXXX AP]
         │
         ▼
[Sensor: HTTP GET http://192.168.4.1/firmware/sensor.bin]
         │
    ┌────┴────────────────────────────────┐
    │ Download OK?                         │
    │ YES                          NO      │
    ▼                              ▼       │
[Write to OTA          [Abort; log error;  │
 partition B]           disconnect STA;    │
    │                   reconnect ESP-NOW] │
    ▼                                      │
[Verify checksum]                          │
    │                                      │
    ├── FAIL ──────────────────────────────┘
    │
    ▼ PASS
[Set boot partition B; reboot]
         │
         ▼
[Boot from B; mark B valid]
         │
         ▼
[Reconnect to hub via ESP-NOW]
         │
         ▼
[Hub web UI: firmware version updated]
```
