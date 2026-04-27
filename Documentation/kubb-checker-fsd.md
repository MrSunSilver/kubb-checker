# Kubb-Checker — Functional Specification Document (FSD)

**Version:** 1.0.0
**Date:** 2026-04-27
**Status:** Draft
**Author:** rolf@dubitzky.de

---

## 1. System Overview

### 1.1 Purpose

Kubb-Checker assists the referee of a Kubb game in objectively determining whether a throw was valid. A throw is valid when the baton rotated at least 360° around its minor axis (end-over-end) **and** rotated vertically — not horizontally like a helicopter throw — between leaving the thrower's hand and hitting a kubb (wooden target block).

### 1.2 Problem Statement

Validating throws by visual inspection is subjective and error-prone, especially at high speed or from a distance. Kubb-Checker embeds an inertial sensor in each baton, measures the throw flight in real time, and streams the verdict to the referee's Android phone via BLE. This eliminates judgment disputes and enforces consistent, objective rules.

### 1.3 Users & Stakeholders

| Role | Responsibility |
|------|---------------|
| Referee / Game Operator | Primary user of the Kubb-Checker-App. Monitors throw validity during play; pairs and calibrates batons before the game. |
| Players | Throw the batons. Affected by the validity judgments but do not operate the system directly. |
| System Administrator | Flashes firmware, programs NFC tags, manages OTA updates. |

### 1.4 Goals & Non-Goals

**Goals:**
- Automatically classify each throw as valid or invalid on both criteria: sufficient rotation (≥ 360° minor-axis) and correct rotation axis (vertical, not helicopter).
- Transmit per-throw event data from each baton to the referee's Android phone via BLE.
- Provide a clear, real-time visual display on the phone for all 6 batons in play.
- Enable sensor calibration, baton naming, and threshold management from the app.
- Support wireless OTA firmware updates for all sensor devices.

**Non-Goals:**
- Automated game scoring, rule enforcement, or game management.
- iOS app support (Android only for Phase 1–6).
- Cloud connectivity, remote data storage, or multi-game history.
- Real-time streaming to spectators or any display other than the referee's phone.
- Audio or visual feedback (LEDs, buzzers) on the baton itself.

### 1.5 High-Level System Flow

```
[Baton thrown]
     │
     ▼
[Kubb-Checker-Sensor: IMU samples throw flight]
     │
     ▼
[Sensor: detect throw start / flight / impact]
[Sensor: compute rotation axes, axis orientation, ToF, height, distance]
     │  BLE Notify (Event-Characteristic)
     ▼
[Kubb-Checker-App (Android): receive and display verdict]
     │
     ▼
[Referee: red/green validity indicators on phone screen]

[Referee: NFC tap on baton] → OOB BLE pairing
[Referee: App → BLE WRITE (Control-Characteristic)] → calibrate / rename / set thresholds / OTA
```

---

## 2. System Architecture

### 2.1 Logical Architecture

The system has two logical tiers:

**Sensor tier** — up to 6 Kubb-Checker-Sensor devices, each embedded in a wooden baton. Each sensor independently detects throws and transmits event and telemetry data via BLE GATT notifications.

**Display & Control tier** — a single Android phone running the Kubb-Checker-App, acting as BLE central. The app maintains up to 6 simultaneous peripheral connections and presents live throw-validity data and baton-management controls to the referee.

Data flows:
- **Sensor → App**: continuous telemetry (10 Hz), throw events (per throw, no loss), status on request or change.
- **App → Sensor**: commands via Control-Characteristic (calibrate, rename, set thresholds, OTA trigger, start/stop).
- **NFC tag → App**: OOB BLE pairing data (tap-to-pair).

### 2.2 Hardware / Platform Architecture

#### 2.2.1 Kubb-Checker-Sensor

| Parameter | Value |
|-----------|-------|
| MCU | ESP32-C3 |
| IMU | Bosch BMI160 (gyroscope + accelerometer) |
| IMU Interface | I2C |
| Form factor | Inside wooden baton, ~300 mm long × ~40 mm diameter |
| Placement | Near center of gravity of the baton |
| NFC tag | NTAG215 affixed to baton exterior |
| Power | LiIon 1S cell (assumed); ADC-based SoC monitoring |
| Quantity per game | ≥ 6 |

**Axis Convention:**
- **Major axis**: Along the baton's length (longitudinal).
- **Minor axis**: Perpendicular to the baton length (transverse / end-over-end).
- A valid throw requires ≥ 360° rotation around the **minor axis**.
- A helicopter throw is detected when the minor rotational axis points predominantly up or down (rather than toward the horizon).

The BMI160 is mounted so that one of its sensor axes aligns with the baton's minor axis. The axis mapping (BMI160 axis ↔ major / minor) is defined as a compile-time constant.

#### 2.2.2 Kubb-Checker-App Host

| Parameter | Value |
|-----------|-------|
| Platform | Android (BLE + NFC capable) |
| BLE role | Central (up to 6 simultaneous peripheral connections) |
| NFC role | Reader (OOB pairing) |

### 2.3 Software Architecture

#### 2.3.1 Sensor Firmware Modules

| Module | Description |
|--------|-------------|
| **IMU Driver** | BMI160 I2C driver; reads raw gyro + accelerometer at ≥ 100 Hz. |
| **Signal Processing** | State machine (Rest / Pre-throw / In-flight / Post-impact / Shaking); gyro integration; throw analysis. |
| **Calibration** | On-command bias/offset computation; sanity check; NVS storage. |
| **BLE (NimBLE)** | GATT server; advertising; 4 characteristics; event queue for zero-loss Event-Characteristic. |
| **NVS** | Persists device name, calibration parameters, thresholds. |
| **OTA** | WiFi HTTP OTA triggered by BLE Control command (assumed); A/B partition; auto-rollback. |
| **Serial Logging** | UART debug output with component tags and timestamps. |

**Sensor Boot Sequence:**
1. Initialize UART; log firmware version.
2. Initialize NVS; load device name, calibration offsets, thresholds (apply defaults if empty).
3. Log loaded NVS values including BLE MAC address and device name to serial.
4. Initialize I2C; detect BMI160 (verify chip ID 0xD1).
5. Apply calibration offsets.
6. Start BLE advertising under stored device name (`kastpinne-<last4hexMAC>` if unconfigured).
7. Enter main loop: IMU sampling → signal processing → BLE notifications.

#### 2.3.2 Kubb-Checker-App Architecture

| Component | Description |
|-----------|-------------|
| **BLE Manager** | BLE central role; up to 6 simultaneous connections; scan, pair, bond, reconnect. |
| **NFC Manager** | Reads NTAG215 OOB record; initiates BLE tap-to-pair. |
| **Data Model (MVVM)** | Per-baton state: connection, last event, telemetry stream, status. (MVVM with Jetpack Compose — assumed.) |
| **Front Page** | Live 6-baton grid; validity indicators; last-thrown highlight. |
| **Settings Sub-page** | Paired baton list; shake identification indicator. |
| **Baton Sub-sub-page** | 3D model, accelerometer bars, calibration trigger, name editing. |
| **Persistence** | Android SharedPreferences or Room DB for pairing history (assumed). |

#### 2.3.3 Sensor Persistence (NVS Namespace: `kubb`)

| NVS Key | Type | Description | Default |
|---------|------|-------------|---------|
| `device_name` | String | Human-readable baton ID | `kastpinne-<MAC[8:12]>` |
| `cal_gyro_x/y/z` | Float | Gyro axis bias offsets (°/s) | 0.0 |
| `cal_accel_x/y/z` | Float | Accelerometer axis bias offsets (m/s²) | 0.0 |
| `thr_throw_accel` | Float | Throw-start translational acceleration threshold (m/s²) | 15.0 |
| `thr_throw_rot` | Float | Throw-start rotation velocity threshold (°/s) | 200.0 |
| `thr_impact` | Float | Impact acceleration threshold (m/s²) | 20.0 |

#### 2.3.4 OTA Update Model

All sensor devices support wireless firmware updates triggered via the BLE Control-Characteristic (`OTA_START` command). Upon receiving the command (including WiFi SSID, password, and firmware URL), the sensor:
1. Connects to the specified WiFi network.
2. Downloads the firmware binary via HTTP.
3. Verifies integrity (ESP-IDF checksum/signature).
4. Writes to the inactive OTA partition (A/B dual-partition scheme).
5. Reboots; on successful boot marks partition valid.
6. If new firmware fails to boot, automatically rolls back to the previous partition.

(WiFi credentials and firmware URL are provided per OTA command via BLE — assumed.)

---

## 3. Implementation Phases

### 3.1 Phase 1 — BLE Connection Foundation

**Scope:**
- Sensor: BLE advertising, GATT server with Telemetry-Characteristic only. Data source is demo-generated (simulated single-axis rotation + constant acceleration in a random direction). No real IMU reads.
- App: BLE scan, NFC tap-to-pair, connect to one baton; display raw Telemetry-Characteristic values as text.

**Deliverables:**
- Sensor firmware v0.1: BLE advertising + demo telemetry stream.
- Android app v0.1: BLE scan/pair/connect + raw telemetry text display.

**Exit Criteria:**
- Sensor visible as `kastpinne-xxxx` in nRF Connect after power-on.
- BLE MAC logged to serial on boot.
- App connects via NFC tap-to-pair; receives continuous telemetry at ~10 Hz.
- Sensor resumes advertising after disconnect.

**Dependencies:**
- NTAG215 tag pre-written with BLE MAC (manual setup step).

---

### 3.2 Phase 2 — Real Gyro Data

**Scope:**
- Sensor: BMI160 I2C driver integration. Replace demo data in Telemetry-Characteristic with live IMU readings. No processing or detection yet — raw calibrated sensor output only.

**Deliverables:**
- Sensor firmware v0.2: Live IMU data via Telemetry-Characteristic.

**Exit Criteria:**
- Stationary baton shows gyro ≈ 0 °/s and accel ≈ 9.8 m/s² on gravity axis.
- Rotating the baton causes corresponding gyro axis to respond.
- No I2C errors in serial log under normal operation.

**Dependencies:**
- Phase 1 complete. BMI160 wired and I2C address confirmed.

---

### 3.3 Phase 3 — Calibration & Settings UI

**Scope:**
- Sensor: Calibration routine via Control-Characteristic; NVS persistence for calibration data, device name, thresholds.
- App: Settings sub-page (single baton entry), Baton sub-sub-page with 3D model, accelerometer bar graphs, calibration button; baton name editing.

**Deliverables:**
- Sensor firmware v0.3: Calibration + full NVS persistence.
- App v0.3: Settings UI, 3D visualization, calibration workflow.

**Exit Criteria:**
- Calibration completes from app; calibrated offsets logged to serial.
- Calibration and device name survive sensor reboot.
- 3D baton model rotates in sync with physical baton movement.
- Sanity-check error returned to app when baton is moving during calibration.

**Dependencies:**
- Phase 2 complete.

---

### 3.4 Phase 4 — Throw Detection & Event Characteristic

**Scope:**
- Sensor: Throw detection algorithm (throw-start, flight phase, impact). Rotation axis analysis (vertical vs. helicopter). Event computation (rotation, ToF, height/distance estimates). Event-Characteristic GATT implementation; no events may be lost.
- App: Front page updated to display Event-Characteristic data; red/green validity indicators for both criteria; highlight of last-thrown baton.

**Deliverables:**
- Sensor firmware v0.4: Throw detection + Event-Characteristic.
- App v0.4: Front page with event display and validity indicators.

**Exit Criteria:**
- 10 test throws (mixed valid/invalid) classified correctly ≥ 90%.
- Helicopter throws classified as invalid on rotation-axis criterion.
- 20 consecutive throws arrive at app with no counter gaps (no event loss).
- Last-thrown baton area highlighted on front page.

**Dependencies:**
- Phase 3 complete; calibration applied.

---

### 3.5 Phase 5 — Status Characteristic & Feature Completeness

**Scope:**
- Sensor: Status-Characteristic (battery %, uptime, calibration state, firmware version); shake detection; remaining control commands (START, STOP, SET_THRESHOLD, OTA_START, FACTORY_RESET).
- App: Status display in baton sub-sub-page.

**Deliverables:**
- Sensor firmware v0.5: Feature-complete.
- App v0.5: Full baton sub-sub-page including status.

**Exit Criteria:**
- Battery percentage displayed and plausible.
- Uptime increments continuously.
- Firmware version matches build tag.
- Shake detection lights up baton indicator in settings sub-page.
- OTA update succeeds; auto-rollback restores previous firmware on bad update.

**Dependencies:**
- Phase 4 complete. ADC battery circuit available on hardware.

---

### 3.6 Phase 6 — Multi-Baton Support

**Scope:**
- App: Full support for 6 simultaneous BLE connections. Front page shows 6 baton areas. Settings sub-page lists all batons. All per-baton features work independently.
- Sensor firmware: No changes expected; verify each baton behaves identically.

**Deliverables:**
- App v1.0: Full multi-baton support (production-ready).

**Exit Criteria:**
- 6 batons connected simultaneously; all telemetry and event streams active.
- Throws from each baton correctly attributed to its area.
- App responsive with 6 active BLE connections (no perceptible UI lag).
- No BLE stability issues with 6 peripherals active.

**Dependencies:**
- Phase 5 complete. 6 assembled, charged, and calibrated sensor units.

---

## 4. Functional & Non-Functional Requirements

### 4.1 Sensor — BLE Communication

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-1.1 | Must | The sensor shall advertise via BLE using the stored device name (default: `kastpinne-<last4hexMAC>`). |
| FR-1.2 | Must | The sensor shall accept BLE connections and resume advertising after disconnection. |
| FR-1.3 | Must | The sensor shall implement a custom GATT service with Event-, Telemetry-, Status-, and Control-Characteristics as specified in Section 6. |
| FR-1.4 | Must | The sensor shall deliver all Event-Characteristic notifications with zero data loss; events shall be queued if the client is temporarily unavailable. |
| FR-1.5 | Must | The sensor shall transmit Telemetry-Characteristic notifications at approximately 10 Hz; sample loss is acceptable. |
| FR-1.6 | Should | The sensor shall transmit Status-Characteristic notifications when values change and respond to READ requests. |
| FR-1.7 | Must | The sensor shall process all Control-Characteristic WRITE commands defined in Section 6.4. |
| FR-1.8 | Must | The sensor shall log its BLE MAC address and device name to the serial console at boot. |
| FR-1.9 | Should | The sensor shall log BLE connection and disconnection events to serial. |

### 4.2 Sensor — IMU Data Acquisition

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-2.1 | Must | The sensor shall initialize the BMI160 via I2C on boot and verify the chip ID (expected: 0xD1). |
| FR-2.2 | Must | The sensor shall sample the BMI160 gyroscope and accelerometer at ≥ 100 Hz during active operation. |
| FR-2.3 | Must | The sensor shall configure the gyroscope full-scale range to ≥ ±2000 °/s and the accelerometer to ≥ ±16 g. |
| FR-2.4 | Must | The sensor shall apply stored calibration offsets to all IMU readings before processing. |
| FR-2.5 | Must | The sensor shall detect and log I2C communication errors. |

### 4.3 Sensor — Gyroscope Calibration

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-3.1 | Must | The sensor shall initiate calibration upon receiving the `CALIBRATE` command via the Control-Characteristic. |
| FR-3.2 | Must | Before calibrating, the sensor shall perform a sanity check: if IMU readings indicate the baton is not at rest, return an error code to the app and abort. |
| FR-3.3 | Must | The sensor shall average IMU bias measurements over a minimum of 2 s to compute calibration parameters. |
| FR-3.4 | Must | The sensor shall store calibration parameters in NVS; they shall survive reboots and power cycles. |
| FR-3.5 | Must | The sensor shall return a success or failure response via the Status-Characteristic upon calibration completion. |

### 4.4 Sensor — Throw Detection

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-4.1 | Must | The sensor shall detect the start of a throw when translational and rotational acceleration exceed configurable thresholds. |
| FR-4.2 | Must | The sensor shall identify the free-flight phase as the period when translational acceleration is within the gravitational band (8.0–10.5 m/s²). |
| FR-4.3 | Must | The sensor shall detect the end of the flight phase (impact) when translational acceleration exceeds the configurable impact threshold. |
| FR-4.4 | Must | The sensor shall compute total rotation around the major axis (°) during the flight phase. |
| FR-4.5 | Must | The sensor shall compute total rotation around the minor axis (°) during the flight phase. |
| FR-4.6 | Must | The sensor shall determine the orientation of the minor rotational axis during flight: if the axis was approximately horizontal (within ±45° of the horizon plane), classify as valid vertical rotation; if pointing predominantly up or down, classify as invalid helicopter throw. |
| FR-4.7 | Must | The sensor shall compute the time-of-flight (s) from throw-start detection to impact detection. |
| FR-4.8 | Should | The sensor shall estimate the peak flight height (m) and horizontal distance (m) by double-integrating translational acceleration during the flight phase. |
| FR-4.9 | Must | The sensor shall transmit an Event-Characteristic notification for each detected throw; no events shall be dropped. |

### 4.5 Sensor — Shake Detection

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-5.1 | Must | The sensor shall detect rapid shaking motion (short-duration high-frequency acceleration pattern) distinct from a throw. |
| FR-5.2 | Must | The sensor shall include a `is_shaking` flag in the Telemetry-Characteristic payload to allow the app to identify the shaken baton. |

### 4.6 Sensor — NVS Persistence

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-6.1 | Must | The sensor shall persist the device name in NVS; the name shall survive reboots and power cycles. |
| FR-6.2 | Must | The sensor shall persist calibration parameters in NVS. |
| FR-6.3 | Must | The sensor shall persist throw-detection threshold values in NVS. |
| FR-6.4 | Must | The sensor shall apply default values for all NVS parameters when NVS is empty or uninitialized. |
| FR-6.5 | Should | The sensor shall log loaded NVS configuration values at startup. |

### 4.7 Sensor — OTA Firmware Update

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-7.1 | Must | The sensor shall support OTA firmware updates triggered via the BLE Control-Characteristic `OTA_START` command. |
| FR-7.2 | Must | The sensor shall verify firmware integrity (checksum/signature) before applying an OTA update. |
| FR-7.3 | Must | The sensor shall use dual OTA partitions (A/B scheme) to enable automatic rollback on boot failure. |
| FR-7.4 | Must | The sensor shall automatically roll back to the previous firmware if the new firmware fails to boot. |
| FR-7.5 | Should | The sensor shall log OTA progress (download %, verification, apply) to the serial console. |

### 4.8 Sensor — NFC Tap-to-Pair

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-8.1 | Must | The sensor shall log its BLE MAC address to serial at boot to enable programming of the NTAG215 NFC tag. |
| FR-8.2 | Must | Each baton's NTAG215 NFC tag shall contain OOB BLE pairing data (BLE MAC address and pairing key) enabling tap-to-pair from the app. |

### 4.9 Sensor — Serial Logging

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-9.1 | Must | The sensor shall log BLE MAC, device name, and firmware version at startup. |
| FR-9.2 | Must | The sensor shall log I2C errors, BLE connection events, calibration results, and throw detection events. |
| FR-9.3 | Should | The sensor shall log OTA progress and results. |
| FR-9.4 | Should | All serial messages shall include a timestamp and a component tag (e.g., `[BLE]`, `[IMU]`, `[FLIGHT]`, `[CAL]`). |

### 4.10 App — BLE Central Management

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-10.1 | Must | The app shall support simultaneous BLE connections to up to 6 Kubb-Checker-Sensor devices. |
| FR-10.2 | Must | The app shall initiate BLE pairing via NFC tap-to-pair (OOB pairing using NTAG215 data). |
| FR-10.3 | Must | The app shall subscribe to Event-, Telemetry-, and Status-Characteristic notifications upon connection. |
| FR-10.4 | Must | The app shall send commands to the sensor via the Control-Characteristic WRITE operation. |
| FR-10.5 | Should | The app shall automatically reconnect to previously paired batons when they come into BLE range. |

### 4.11 App — Front Page

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-11.1 | Must | The front page shall display the title "Kubb Checker" centered at the top. |
| FR-11.2 | Must | The front page shall show six equal framed areas, one per baton slot; unused slots shall appear empty. |
| FR-11.3 | Must | Each connected baton area shall display the baton ID, last throw statistics summary, and a motion-state indicator (moving / resting). |
| FR-11.4 | Must | Each baton area shall show two red/green validity indicators: (1) vertical rotation (not helicopter), (2) at least one full rotation (≥ 360°). |
| FR-11.5 | Must | The baton area of the most recently thrown baton shall be highlighted with a distinct frame. |
| FR-11.6 | Must | A settings button (gear icon) shall be displayed in the upper-right corner of the front page. |

### 4.12 App — Settings Sub-Page

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-12.1 | Must | The settings sub-page shall list all paired batons showing BLE MAC address, device name, and connection status. |
| FR-12.2 | Must | Each baton entry shall display a shake/rapid-movement indicator to allow physical identification of a specific baton. |
| FR-12.3 | Must | Selecting a baton entry shall navigate to that baton's sub-sub-page. |

### 4.13 App — Baton Sub-Sub-Page

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-13.1 | Must | The baton sub-sub-page shall display the baton ID at the top. |
| FR-13.2 | Must | The page shall render a 3D model of a wooden cylinder with slightly rounded edges that rotates in real time according to Telemetry-Characteristic gyro readings. |
| FR-13.3 | Must | Three bar graphs shall display translational acceleration in X, Y, and Z axes on a scale of 0–100 m/s². |
| FR-13.4 | Must | A "Calibrate" button shall trigger the gyroscope calibration procedure on the connected baton. |
| FR-13.5 | Must | The app shall display step-by-step calibration guidance (instruction to lay baton flat and still) and show success or error feedback. |

### 4.14 App — Baton Name Editing

| ID | Priority | Requirement |
|----|----------|-------------|
| FR-14.1 | Must | An "Edit" button next to the baton ID shall enable inline editing of the device name. |
| FR-14.2 | Must | After editing is confirmed, the app shall write the new name to the baton via the Control-Characteristic `SET_NAME` command. |
| FR-14.3 | Must | The baton shall persist the new name in NVS and use it in BLE advertising after the next reboot. |

### 4.15 Non-Functional Requirements

| ID | Priority | Requirement |
|----|----------|-------------|
| NFR-1.1 | Must | The IMU shall be sampled at ≥ 100 Hz to achieve sufficient temporal resolution for throw detection. |
| NFR-1.2 | Must | Throw validity classification shall achieve ≥ 90% accuracy across a defined set of valid and invalid reference throws. |
| NFR-1.3 | Must | Event-Characteristic notifications shall be delivered with zero data loss; events shall be queued when BLE is temporarily unavailable. |
| NFR-1.4 | Should | BLE telemetry notification latency shall be ≤ 100 ms end-to-end. |
| NFR-1.5 | Must | The sensor electronics and battery shall physically fit inside the baton housing (≤ 300 mm × 40 mm diameter). |
| NFR-1.6 | Should | The sensor shall survive the mechanical shock of a typical Kubb throw and impact (estimated peak acceleration ≥ 50 m/s² — assumed). |
| NFR-2.1 | Must | All 6 baton sensors shall remain operational throughout a full game session (assumed ≥ 2 hours). |
| NFR-2.2 | Should | The app shall remain responsive (no perceptible UI lag) with 6 simultaneous active BLE connections and continuous telemetry. |
| NFR-3.1 | Should | Calibration parameters and device names shall persist through ≥ 1000 NVS write cycles without corruption. |
| NFR-3.2 | Must | OTA firmware integrity shall be verified before flashing; corrupt or mismatched binaries shall be rejected. |
| NFR-4.1 | Should | The 3D baton model in the app shall render at ≥ 30 fps when telemetry is active. |
| NFR-5.1 | Should | The BLE MAC address shall be stable (non-randomized) across reboots to ensure consistent NFC tap-to-pair. |

### 4.16 Constraints

| ID | Constraint |
|----|-----------|
| CON-1 | Sensor firmware shall target ESP-IDF (or Arduino framework on ESP-IDF) for the ESP32-C3. |
| CON-2 | App platform is Android only; minimum API level TBD (assumed API 26 / Android 8.0). |
| CON-3 | Sensor PCB and LiIon 1S battery shall fit within baton dimensions (≤ 300 mm × 40 mm diameter). |
| CON-4 | Throw validity criteria (≥ 360° minor-axis rotation, vertical rotation axis) are fixed by Kubb rules and are not end-user configurable. |
| CON-5 | BLE MAC address must be public and stable (disable address randomization in ESP-IDF BLE config). |
| CON-6 | BMI160 I2C address is 0x68 (SDO = GND) unless hardware requires 0x69 (SDO = VDD). |
| CON-7 | OTA partition scheme requires ≥ 2 MB flash on each ESP32-C3 module. |

---

## 5. Risks, Assumptions & Dependencies

| # | Risk / Assumption | Type | Likelihood | Impact | Mitigation |
|---|-------------------|------|------------|--------|-----------|
| R-1 | Double integration of acceleration for height/distance estimation produces unacceptable drift | Technical risk | High | Low | Report height/distance as best-effort only; mark in UI. Fall back to time-of-flight only if drift is unacceptable. FR-4.8 is Should-priority. |
| R-2 | Android device may not support 6 simultaneous BLE central connections | Technical risk | Medium | High | Test on target hardware early in Phase 6. Document minimum Android version and BLE capability. Provide graceful degradation for fewer simultaneous connections. |
| R-3 | Mechanical shock from throws and impacts may damage sensor PCB or loosen I2C connections | Physical risk | Medium | High | Use conformal coating and secure mounting. Validate with 50-throw soak test (AT-004). |
| R-4 | Battery life may be insufficient for a 2-hour session | Technical risk | Medium | Medium | Implement IMU duty-cycling and BLE connection-interval optimization between throws. |
| R-5 | Throw detection thresholds require per-player or per-field tuning | Design assumption | High | Low | Expose all thresholds as configurable NVS parameters adjustable via BLE Control command. |
| R-6 | Rotation axis orientation algorithm may misclassify borderline throws | Algorithm risk | Medium | Medium | Define clear axis-angle threshold (±45° from horizon). Validate on ≥ 30 reference throws. |
| A-1 | OTA update uses WiFi HTTP download triggered via BLE Control command; WiFi credentials provided per command. (assumed) | Assumption | — | — | Confirm OTA mechanism before Phase 5. Alternative: BLE DFU via NimBLE MCUboot. |
| A-2 | Battery type is LiIon 1S with integrated charging circuit; level measured via ESP32-C3 ADC. (assumed) | Assumption | — | — | Confirm hardware design before Phase 5. |
| A-3 | IMU sampling at 100 Hz is sufficient for accurate rotation detection at typical throw speeds. (assumed) | Assumption | — | — | Verify with real throw data in Phase 4. |
| A-4 | App uses MVVM architecture with Jetpack Compose for Android UI. (assumed) | Assumption | — | — | Confirm framework before Phase 1 app development. |
| A-5 | BLE MAC address on ESP32-C3 is public and stable across reboots (address randomization disabled). (assumed) | Assumption | — | — | Verify in ESP-IDF BLE configuration before Phase 1. |
| A-6 | Throw timestamps are milliseconds since boot; no RTC or absolute wall-clock time available. (assumed) | Assumption | — | — | Document limitation; add RTC if absolute time is later required. |
| D-1 | NTAG215 tags must be programmed with BLE MAC before first pairing (manual setup step) | Dependency | — | — | Document tag-writing procedure in operational procedures. |
| D-2 | 6 fully assembled, battery-equipped sensor units required for Phase 6 testing | Dependency | — | — | Track hardware assembly timeline. |

---

## 6. Interface Specifications

### 6.1 BLE GATT Profile

#### 6.1.1 Custom Kubb-Checker Service

| Attribute | Value |
|-----------|-------|
| Service Name | Kubb-Checker Service |
| Service UUID | `F4B20000-3E1A-4B7C-9A8D-2C6E1F0A5B3D` (placeholder — finalize during Phase 1) |

#### 6.1.2 Characteristic Summary

| Characteristic | UUID | Properties | Description |
|---------------|------|------------|-------------|
| Event | `F4B20001-3E1A-4B7C-9A8D-2C6E1F0A5B3D` | NOTIFY | Per-throw event; zero-loss queued delivery. |
| Telemetry | `F4B20002-3E1A-4B7C-9A8D-2C6E1F0A5B3D` | NOTIFY | Continuous 10 Hz IMU stream; loss acceptable. |
| Status | `F4B20003-3E1A-4B7C-9A8D-2C6E1F0A5B3D` | READ + NOTIFY | Battery, uptime, calibration state, firmware version. |
| Control | `F4B20004-3E1A-4B7C-9A8D-2C6E1F0A5B3D` | WRITE | Commands from app to sensor. |

### 6.2 Internal Interfaces

#### 6.2.1 BMI160 I2C Interface

| Parameter | Value |
|-----------|-------|
| Protocol | I2C |
| Clock frequency | 400 kHz (Fast Mode) |
| I2C address | 0x68 (SDO = GND) |
| Chip ID register | 0x00, expected value 0xD1 |
| Sample rate | ≥ 100 Hz (ODR register ACC_CONF / GYR_CONF) |
| Gyro full-scale | ±2000 °/s (GYR_RANGE = 0x00) |
| Accel full-scale | ±16 g (ACC_RANGE = 0x0C) |

### 6.3 Data Models / Payload Schemas

All payloads are little-endian. MTU negotiation required for Event-Characteristic (39 bytes exceeds default 20-byte MTU; negotiate ≥ 64-byte MTU during connection setup).

#### 6.3.1 Event-Characteristic Payload (39 bytes)

| Field | Type | Bytes | Unit | Description |
|-------|------|-------|------|-------------|
| `event_counter` | uint16 | 2 | # | Monotonically increasing per-throw counter |
| `timestamp_ms` | uint32 | 4 | ms | Time since boot at impact detection |
| `rotation_major` | float32 | 4 | ° | Total rotation around major axis during flight |
| `rotation_minor` | float32 | 4 | ° | Total rotation around minor axis during flight |
| `time_of_flight` | float32 | 4 | s | Flight duration (throw-start to impact) |
| `height_m` | float32 | 4 | m | Estimated peak flight height (0 if not computed) |
| `distance_m` | float32 | 4 | m | Estimated horizontal distance (0 if not computed) |
| `max_trans_accel` | int16 | 2 | 0.01 m/s² | Maximum translational acceleration during flight |
| `min_trans_accel` | int16 | 2 | 0.01 m/s² | Minimum translational acceleration during flight |
| `avg_trans_accel` | int16 | 2 | 0.01 m/s² | Average translational acceleration during flight |
| `max_rot_accel` | int16 | 2 | 0.1 °/s² | Maximum rotational acceleration during flight |
| `min_rot_accel` | int16 | 2 | 0.1 °/s² | Minimum rotational acceleration during flight |
| `avg_rot_accel` | int16 | 2 | 0.1 °/s² | Average rotational acceleration during flight |
| `flags` | uint8 | 1 | — | Bit 0: 1=vertical rotation (valid), 0=helicopter (invalid); Bit 1: 1=≥360° rotation (valid), 0=insufficient |

**Total: 39 bytes**

#### 6.3.2 Telemetry-Characteristic Payload (18 bytes, 10 Hz)

| Field | Type | Bytes | Unit | Description |
|-------|------|-------|------|-------------|
| `timestamp_ms` | uint32 | 4 | ms | Time since boot |
| `gyro_x` | int16 | 2 | 0.01 °/s | Calibrated gyroscope X |
| `gyro_y` | int16 | 2 | 0.01 °/s | Calibrated gyroscope Y |
| `gyro_z` | int16 | 2 | 0.01 °/s | Calibrated gyroscope Z |
| `accel_x` | int16 | 2 | 0.01 m/s² | Calibrated accelerometer X |
| `accel_y` | int16 | 2 | 0.01 m/s² | Calibrated accelerometer Y |
| `accel_z` | int16 | 2 | 0.01 m/s² | Calibrated accelerometer Z |
| `motion_state` | uint8 | 1 | enum | 0=rest, 1=pre-throw, 2=in-flight, 3=post-impact, 4=shaking |
| `is_shaking` | uint8 | 1 | bool | 1 if shake detected |

**Total: 18 bytes**

#### 6.3.3 Status-Characteristic Payload (10 bytes)

| Field | Type | Bytes | Unit | Description |
|-------|------|-------|------|-------------|
| `battery_pct` | uint8 | 1 | % | Battery state-of-charge 0–100 |
| `uptime_s` | uint32 | 4 | s | Seconds since boot |
| `cal_status` | uint8 | 1 | enum | 0=uncalibrated, 1=calibrated, 2=in-progress, 3=error |
| `fw_major` | uint8 | 1 | — | Firmware major version |
| `fw_minor` | uint8 | 1 | — | Firmware minor version |
| `fw_patch` | uint8 | 1 | — | Firmware patch version |
| `reserved` | uint8 | 1 | — | Reserved; shall be 0 |

**Total: 10 bytes**

### 6.4 Control-Characteristic Commands (App → Sensor)

All commands are WRITE (no response) or WRITE WITH RESPONSE. The first byte is the opcode.

| Opcode | Command | Payload | Description |
|--------|---------|---------|-------------|
| `0x01` | `CALIBRATE` | none | Start gyro calibration. Result reported via Status-Characteristic `cal_status`. |
| `0x02` | `SET_NAME` | `[len: uint8][name: utf8, ≤20 chars]` | Set device name persistently in NVS. Advertising updated on next reboot. |
| `0x03` | `START` | none | Resume IMU sampling and BLE notifications. |
| `0x04` | `STOP` | none | Pause IMU sampling and BLE notifications (low-power). |
| `0x05` | `SET_THRESHOLD` | `[id: uint8][value: float32]` | Update a throw-detection threshold in NVS. Threshold IDs in Appendix 10.2. |
| `0x06` | `OTA_START` | `[ssid_len: uint8][ssid][pwd_len: uint8][pwd][url: utf8 remainder]` | Trigger WiFi OTA. Sensor connects to WiFi and downloads firmware from URL. |
| `0x07` | `FACTORY_RESET` | none | Erase all NVS data and reboot. |

---

## 7. Operational Procedures

### 7.1 Initial Setup (Pre-Game)

1. **Assemble hardware**: Insert charged sensor into baton; secure near center of gravity.
2. **Flash firmware**: Flash sensor firmware via USB: `idf.py flash` (initial flash only; subsequent updates via OTA).
3. **Record BLE MAC**: Open serial monitor; note `BLE MAC: XX:XX:XX:XX:XX:XX` logged at boot.
4. **Program NFC tag**: Write BLE MAC and OOB pairing data to the NTAG215 tag using an NFC tag writer app (e.g., NFC Tools for Android). Affix tag to baton exterior.
5. **Repeat** steps 1–4 for all 6 sensors.
6. **Pair batons to app**: Open Kubb-Checker-App; tap phone to each baton's NFC tag; confirm BLE pairing prompt.
7. **Calibrate**: For each baton: Settings → select baton → lay baton flat on stable surface → press "Calibrate" → wait for success confirmation.
8. **Verify**: Check front page: all 6 baton areas connected, at-rest state, no pending throw events.

### 7.2 Normal Gameplay

1. Confirm all batons show "connected" status on the front page.
2. Referee monitors the front page during throws.
3. After each throw: the thrown baton's area highlights; red/green validity indicators update within ~1 s of impact.
4. Referee announces validity based on the two indicators.
5. Between throws: indicators show the last throw result until the next throw is detected.

### 7.3 Baton Identification

1. Open Settings sub-page.
2. Ask the player to shake the baton rapidly.
3. The corresponding baton entry in the list shows the shake indicator.

### 7.4 Recalibration During a Game

1. In Settings, select the baton requiring recalibration.
2. Retrieve the baton; lay it flat on a stable, level surface.
3. Press "Calibrate" in the app; confirm success before returning the baton to play.

### 7.5 Firmware OTA Update

1. Build and host the new firmware binary on an HTTP server accessible via WiFi.
2. In the app (Phase 5 UI TBD), trigger `OTA_START` for the target baton, providing WiFi SSID, password, and firmware URL.
3. Monitor serial console for OTA progress.
4. Sensor reboots and applies new firmware; app reconnects automatically.

> **Note:** The sensor does not detect throws during OTA. Complete active game rounds before updating.

### 7.6 Factory Reset

1. Send `FACTORY_RESET` command via the Control-Characteristic from the app.
2. Sensor erases all NVS and reboots with factory defaults.
3. Re-pair and recalibrate as per Section 7.1.

### 7.7 Recovery Procedures

| Scenario | Action |
|----------|--------|
| Baton not visible in BLE scan | Check battery; power-cycle sensor; verify NFC tag was programmed correctly. |
| Calibration sanity-check error | Ensure baton is completely at rest on a flat surface; retry. Check for nearby vibration sources. |
| OTA failed / rollback triggered | Check serial log for error; re-attempt OTA with a verified firmware binary. |
| App loses connection mid-game | App auto-reconnects; Event-Characteristic queue retains events for delivery on reconnect. |
| Sensor crashes / watchdog reset | Sensor reboots and resumes advertising; reconnect from app. |

---

## 8. Verification & Validation

### 8.1 Phase 1 Verification — BLE Connection

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P1-001 | BLE advertising | Power on sensor; scan with nRF Connect | `kastpinne-xxxx` visible within 5 s |
| TC-P1-002 | BLE MAC serial log | Open serial monitor; power on sensor | `BLE MAC: XX:XX:XX:XX:XX:XX` logged at boot |
| TC-P1-003 | NFC tap-to-pair | Tap phone to NFC tag | BLE pairing prompt appears in app |
| TC-P1-004 | BLE connection | Accept pairing | Connection established; GATT services enumerated |
| TC-P1-005 | Telemetry stream | Subscribe to Telemetry-Characteristic | Notifications received at ~10 Hz |
| TC-P1-006 | Demo data content | Inspect telemetry payload | Gyro and accel fields contain demo-generated values |
| TC-P1-007 | Reconnect after disconnect | Disconnect; wait 5 s | Sensor resumes advertising; app reconnects |

### 8.2 Phase 2 Verification — Real Gyro Data

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P2-001 | Stationary IMU | Hold baton still; observe telemetry | Gyro ≈ 0 °/s; accel ≈ 9.8 m/s² on gravity axis |
| TC-P2-002 | Rotation detection | Rotate baton ~90° around each axis | Corresponding gyro axis responds with correct sign |
| TC-P2-003 | Acceleration detection | Move baton linearly | Corresponding accel axis changes magnitude |
| TC-P2-004 | I2C error handling | Disconnect BMI160 SDA/SCL; boot | `[IMU] I2C error` logged; no crash or hang |

### 8.3 Phase 3 Verification — Calibration & Settings

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P3-001 | Calibration success | Lay baton flat; trigger calibration from app | `[CAL] success` on serial; status = 1 (calibrated) |
| TC-P3-002 | Calibration sanity check | Move baton during calibration trigger | Error code returned to app; calibration aborted |
| TC-P3-003 | Calibration persistence | Calibrate; power-cycle sensor; check serial | Calibration values logged from NVS at boot |
| TC-P3-004 | Baton name change | Enter new name in app; confirm | Control command sent; serial confirms NVS write |
| TC-P3-005 | Name persistence | Change name; power-cycle; scan BLE | Advertising uses new name |
| TC-P3-006 | 3D model synchronization | Rotate baton; observe 3D model in app | Model rotates matching physical movement in real time |
| TC-P3-007 | Accelerometer bar graphs | Move baton along each axis | Corresponding bar graph responds proportionally |

### 8.4 Phase 4 Verification — Throw Detection

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P4-001 | Throw detection | Perform 10 throws | Event notification received for each throw; no missed events |
| TC-P4-002 | Throw counter monotonicity | 10 throws; check `event_counter` | Values 1–10 in order; no gaps |
| TC-P4-003 | Rotation measurement | Throw with known ~360° rotation | `rotation_minor` reported within ±30° of reference |
| TC-P4-004 | Valid throw classification | 10 vertical throws (end-over-end) | `flags` bit 0 = 1 for all valid throws |
| TC-P4-005 | Helicopter throw detection | 5 helicopter-style throws | `flags` bit 0 = 0 for all helicopter throws |
| TC-P4-006 | Sufficient rotation flag | Throw with confirmed ≥ 360° | `flags` bit 1 = 1 |
| TC-P4-007 | Insufficient rotation flag | Partial throw (< 360°) | `flags` bit 1 = 0 |
| TC-P4-008 | Time-of-flight | Throw with simultaneous manual timing | `time_of_flight` within ±0.2 s of reference |
| TC-P4-009 | Front page validity indicators | Valid throw | Both indicators show green on front page |
| TC-P4-010 | Front page helicopter indicator | Helicopter throw | Rotation-axis indicator shows red; rotation indicator independent |
| TC-P4-011 | Last-thrown baton highlight | Throw baton 2 of 2 connected | Baton 2 area highlighted on front page |
| TC-P4-012 | No event data loss | 20 consecutive throws | `event_counter` increments without gaps |

### 8.5 Phase 5 Verification — Status & Feature Completeness

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P5-001 | Battery level | Read Status-Characteristic | Battery % displayed; value plausible for charge level |
| TC-P5-002 | Uptime counter | Read Status twice 60 s apart | `uptime_s` increments by ~60 |
| TC-P5-003 | Firmware version | Read Status | Version matches build tag (e.g., `1.0.0`) |
| TC-P5-004 | Calibration status | Read Status before and after calibration | `cal_status` transitions 0 → 1 after successful calibration |
| TC-P5-005 | Shake detection | Shake baton rapidly | `is_shaking` = 1 in telemetry; shake indicator visible in app settings |
| TC-P5-006 | STOP command | Send STOP via Control | IMU and BLE notifications pause; no crash |
| TC-P5-007 | START command | Send START after STOP | Notifications resume; normal operation |
| TC-P5-008 | OTA update | Trigger OTA with valid firmware URL | Firmware updated; `fw_major/minor/patch` updated in Status |
| TC-P5-009 | OTA rollback | Flash intentionally bad firmware via OTA | Sensor reboots; old version restored; confirmed via Status |

### 8.6 Phase 6 Verification — Multi-Baton

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-P6-001 | 6 simultaneous connections | Connect 6 batons | All 6 show connected status on front page |
| TC-P6-002 | 6-baton telemetry | Observe front page | All 6 areas show live motion-state updates |
| TC-P6-003 | Concurrent throw events | Throw 2 batons within 1 s | Both events received; correctly attributed to each baton area |
| TC-P6-004 | App responsiveness | 6 batons active; navigate all pages | No perceptible UI lag on front page or settings |
| TC-P6-005 | Baton identification (shake) | Shake baton 3; observe settings | Baton 3 entry shows shake indicator; others do not |

### 8.7 Acceptance Tests

| Test ID | Scenario | Procedure | Success Criteria |
|---------|----------|-----------|-----------------|
| AT-001 | Full game simulation | 6 batons; simulate 30 min of play (~30 throws) | All events received; no crashes; validity classifications correct |
| AT-002 | Classification accuracy | 30 confirmed-valid + 30 confirmed-invalid reference throws | ≥ 90% classification accuracy on both criteria |
| AT-003 | Battery endurance | Fully charge all 6 batons; run 2-hour game session | All batons operational at session end |
| AT-004 | Mechanical durability | Throw each baton 50 times onto grass and wooden blocks | No sensor malfunction; I2C stable; no crashes |
| AT-005 | BLE range | Deploy across full Kubb field (~8 m × 5 m) | All 6 BLE connections stable across full field area |
| AT-006 | UI layout — front page | Connect 6 batons; observe front page | Title centered; 6 equal areas; validity indicators; gear button present |
| AT-007 | UI layout — settings | Navigate to settings; select baton | Baton list shows MAC/name; sub-sub-page opens with 3D model |
| AT-008 | Physical fit | Assemble full sensor + battery into baton | Electronics fit entirely within baton; baton closes and is throwable |

### 8.8 BLE Standard Test Cases

The following tests are drawn from the BLE standard test specification and apply to the Kubb-Checker-Sensor.

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-BLE-100 | Discovery and connection | Power on; scan; connect; enumerate services; disconnect | All steps succeed; advertising resumes after disconnect |
| TC-BLE-102 | GATT read/write | Read Status; write Control; read back; write invalid data | All GATT operations behave per characteristic properties; invalid write returns error |
| TC-BLE-103 | Notification latency | Subscribe to Telemetry; trigger data change; measure latency | Latency ≤ 100 ms; 10 rapid changes all received in order |
| EC-BLE-200 | Disconnect during transfer | Connect; begin large payload; force disconnect | No crash; device resumes advertising; reconnect succeeds |
| EC-BLE-201 | Rapid connect/disconnect cycles | 20 rapid connect/disconnect cycles | No memory leak; device remains functional |
| EC-BLE-204 | Bonding persistence | Pair and bond; reboot device; reconnect | Reconnects without re-pairing; factory reset clears bonding |

### 8.9 NVS Standard Test Cases

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-NVS-100 | Config persistence across reboot | Write name and calibration; reboot | Values match after reboot and power cycle |
| TC-NVS-101 | Default values on first boot | Erase NVS; boot device | Default device name used; no crash; defaults logged |
| EC-NVS-200 | NVS corruption recovery | Corrupt NVS partition; boot | Falls back to defaults; no crash |
| EC-NVS-202 | Power loss during NVS write | Cut power during calibration save | Either old or new value present; no corruption |

### 8.10 OTA Standard Test Cases

| Test ID | Feature | Procedure | Success Criteria |
|---------|---------|-----------|-----------------|
| TC-OTA-100 | Successful OTA update | Upload firmware V2; trigger OTA | V2 running after reboot; all features operational |
| TC-OTA-101 | OTA rollback on bad firmware | Flash intentionally broken firmware | Sensor rolls back to V1 automatically; no manual action needed |
| EC-OTA-202 | Invalid firmware rejection | Attempt OTA with random binary | Rejected — checksum/magic fail; original firmware intact |

### 8.11 Traceability Matrix

| Requirement | Priority | Test Case(s) | Status |
|------------|----------|-------------|--------|
| FR-1.1 | Must | TC-P1-001, TC-P3-005 | Covered |
| FR-1.2 | Must | TC-P1-007 | Covered |
| FR-1.3 | Must | TC-P1-004 | Covered |
| FR-1.4 | Must | TC-P4-012 | Covered |
| FR-1.5 | Must | TC-P1-005, TC-P1-006 | Covered |
| FR-1.6 | Should | TC-P5-001, TC-P5-002, TC-P5-003 | Covered |
| FR-1.7 | Must | TC-P3-001, TC-P5-006, TC-P5-007, TC-P5-008 | Covered |
| FR-1.8 | Must | TC-P1-002 | Covered |
| FR-1.9 | Should | TC-P1-007 | Covered |
| FR-2.1 | Must | TC-P2-001, TC-P2-004 | Covered |
| FR-2.2 | Must | TC-P2-001, TC-P2-002, TC-P2-003 | Covered |
| FR-2.3 | Must | TC-P2-001, TC-P2-002 | Covered |
| FR-2.4 | Must | TC-P3-001, TC-P3-003 | Covered |
| FR-2.5 | Must | TC-P2-004 | Covered |
| FR-3.1 | Must | TC-P3-001 | Covered |
| FR-3.2 | Must | TC-P3-002 | Covered |
| FR-3.3 | Must | TC-P3-001 | Covered |
| FR-3.4 | Must | TC-P3-003 | Covered |
| FR-3.5 | Must | TC-P3-001, TC-P3-002, TC-P5-004 | Covered |
| FR-4.1 | Must | TC-P4-001 | Covered |
| FR-4.2 | Must | TC-P4-001 | Covered |
| FR-4.3 | Must | TC-P4-001 | Covered |
| FR-4.4 | Must | TC-P4-003 | Covered |
| FR-4.5 | Must | TC-P4-003, TC-P4-006, TC-P4-007 | Covered |
| FR-4.6 | Must | TC-P4-004, TC-P4-005, AT-002 | Covered |
| FR-4.7 | Must | TC-P4-008 | Covered |
| FR-4.8 | Should | TC-P4-008 | Partially Covered |
| FR-4.9 | Must | TC-P4-012 | Covered |
| FR-5.1 | Must | TC-P5-005 | Covered |
| FR-5.2 | Must | TC-P5-005 | Covered |
| FR-6.1 | Must | TC-P3-004, TC-P3-005, TC-NVS-100 | Covered |
| FR-6.2 | Must | TC-P3-003, TC-NVS-100 | Covered |
| FR-6.3 | Must | TC-P3-003 | Covered |
| FR-6.4 | Must | TC-NVS-101 | Covered |
| FR-6.5 | Should | TC-P1-002 | Covered |
| FR-7.1 | Must | TC-P5-008 | Covered |
| FR-7.2 | Must | EC-OTA-202 | Covered |
| FR-7.3 | Must | TC-P5-009, TC-OTA-101 | Covered |
| FR-7.4 | Must | TC-P5-009, TC-OTA-101 | Covered |
| FR-7.5 | Should | TC-P5-008 | Covered |
| FR-8.1 | Must | TC-P1-002 | Covered |
| FR-8.2 | Must | TC-P1-003 | Covered |
| FR-9.1 | Must | TC-P1-002 | Covered |
| FR-9.2 | Must | TC-P2-004, TC-P3-001, TC-P4-001 | Covered |
| FR-9.3 | Should | TC-P5-008 | Covered |
| FR-9.4 | Should | TC-P1-002 | Covered |
| FR-10.1 | Must | TC-P6-001 | Covered |
| FR-10.2 | Must | TC-P1-003, TC-P1-004 | Covered |
| FR-10.3 | Must | TC-P1-005 | Covered |
| FR-10.4 | Must | TC-P3-001, TC-P5-006, TC-P5-007 | Covered |
| FR-10.5 | Should | TC-P1-007 | Covered |
| FR-11.1 | Must | AT-006 | Covered |
| FR-11.2 | Must | TC-P6-001, AT-006 | Covered |
| FR-11.3 | Must | TC-P4-009, TC-P4-010 | Covered |
| FR-11.4 | Must | TC-P4-009, TC-P4-010 | Covered |
| FR-11.5 | Must | TC-P4-011 | Covered |
| FR-11.6 | Must | AT-006 | Covered |
| FR-12.1 | Must | AT-007 | Covered |
| FR-12.2 | Must | TC-P5-005 | Covered |
| FR-12.3 | Must | AT-007 | Covered |
| FR-13.1 | Must | AT-007 | Covered |
| FR-13.2 | Must | TC-P3-006 | Covered |
| FR-13.3 | Must | TC-P3-007 | Covered |
| FR-13.4 | Must | TC-P3-001 | Covered |
| FR-13.5 | Must | TC-P3-001, TC-P3-002 | Covered |
| FR-14.1 | Must | TC-P3-004 | Covered |
| FR-14.2 | Must | TC-P3-004 | Covered |
| FR-14.3 | Must | TC-P3-005 | Covered |
| NFR-1.1 | Must | TC-P2-001 | Covered |
| NFR-1.2 | Must | AT-002 | Covered |
| NFR-1.3 | Must | TC-P4-012 | Covered |
| NFR-1.4 | Should | TC-BLE-103 | Covered |
| NFR-1.5 | Must | AT-008 | Covered |
| NFR-1.6 | Should | AT-004 | Covered |
| NFR-2.1 | Must | AT-003 | Covered |
| NFR-2.2 | Should | TC-P6-004 | Covered |
| NFR-3.1 | Should | EC-NVS-202, TC-NVS-100 | Covered |
| NFR-3.2 | Must | EC-OTA-202, TC-P5-009 | Covered |
| NFR-4.1 | Should | TC-P3-006 | Covered |
| NFR-5.1 | Should | TC-P1-002, TC-P1-003 | Covered |

**GAP Summary:**

| Requirement | Gap Reason |
|-------------|-----------|
| FR-4.8 | Double-integration height/distance is best-effort; partial coverage via TC-P4-008 (time-of-flight test). Dedicated height/distance accuracy test deferred until Phase 4 field evaluation. |

---

## 9. Troubleshooting Guide

| Symptom | Likely Cause | Diagnostic Steps | Corrective Action |
|---------|-------------|-----------------|-------------------|
| Sensor not visible in BLE scan | Battery dead; BLE init failed | Check serial log at boot; measure battery voltage | Charge/replace battery; reflash firmware |
| NFC tap-to-pair not working | NFC tag not programmed or wrong MAC | Read tag with NFC tool; compare MAC to serial log | Reprogram NFC tag with correct BLE MAC |
| App connects but no telemetry | CCCD not written; MTU too small | Inspect with nRF Connect; check CCCD subscription | Re-establish connection; verify MTU negotiation ≥ 64 bytes |
| Telemetry shows all zeros | BMI160 I2C failure | Check serial for `[IMU] I2C error` | Re-solder I2C pins; verify BMI160 address (0x68 vs 0x69) |
| Calibration always fails sanity check | Baton not truly at rest; surface vibration | Move to isolated stable surface; check serial for variance values | Increase sanity-check tolerance; check `thr_throw_accel` NVS value |
| All throws classified invalid — helicopter | Minor-axis orientation algorithm incorrect; wrong axis mapping | Log rotation axis orientation vector during throws to serial | Verify body-frame vs. world-frame axis mapping in firmware |
| All throws classified invalid — rotation | Calibration drift; minor-axis mapped to wrong BMI160 axis | Check `rotation_minor` vs `rotation_major` on serial — values swapped? | Swap axis mapping constant; recalibrate |
| Events received but `event_counter` has gaps | BLE disconnection while event was queued; queue overflow | Check serial for `[BLE] event queued/dropped` | Verify event queue size is sufficient; ensure reliable BLE connection |
| OTA fails immediately | WiFi credentials wrong; URL unreachable | Check serial for WiFi connect and HTTP error codes | Correct SSID/password; verify firmware server URL |
| OTA applied but sensor boots old firmware | New firmware crashes at boot; watchdog triggers rollback | Check serial crash log before rollback message | Fix boot issue in new firmware; reflash via USB if needed |
| App crashes with 6 batons connected | Android BLE connection limit; memory leak | Check logcat for BLE errors; check device BLE capability | Test with fewer connections; verify target device supports 6 BLE centrals |
| 3D model lags or stutters | BLE receive thread blocking render thread | Profile app; check telemetry receive rate | Decouple BLE receive from render thread using ring buffer |
| Battery SoC always 0% | ADC pin misconfigured; voltage divider wrong | Check serial for raw ADC value | Verify hardware voltage divider ratio; adjust scaling constant |

---

## 10. Appendix

### 10.1 Configuration Defaults

| Parameter | Default | Notes |
|-----------|---------|-------|
| IMU sample rate | 100 Hz | BMI160 ODR configuration |
| Telemetry rate | 10 Hz | Subsampled from IMU |
| Throw-start accel threshold | 15.0 m/s² | NVS key `thr_throw_accel`; SET_THRESHOLD ID 0x01 |
| Throw-start rotation threshold | 200.0 °/s | NVS key `thr_throw_rot`; SET_THRESHOLD ID 0x02 |
| Impact threshold | 20.0 m/s² | NVS key `thr_impact`; SET_THRESHOLD ID 0x03 |
| Gravitational band min | 8.0 m/s² | SET_THRESHOLD ID 0x04 |
| Gravitational band max | 10.5 m/s² | SET_THRESHOLD ID 0x05 |
| Calibration averaging window | 2.0 s | Duration of bias averaging |
| Calibration sanity threshold | 2.0 m/s² | Max variance from gravity during sanity check |
| Min valid minor-axis rotation | 360.0 ° | Fixed by Kubb rules; not user-configurable |
| Max helicopter axis angle | 45.0 ° | Angle from horizon; beyond this → helicopter |
| Default device name prefix | `kastpinne-` | Followed by last 4 hex digits of BLE MAC |
| NVS namespace | `kubb` | All sensor NVS keys |
| Serial baud rate | 115200 | UART0 |
| BLE advertise interval | 100 ms (assumed) | |

### 10.2 SET_THRESHOLD Opcode IDs

| ID | Parameter |
|----|-----------|
| 0x01 | Throw-start translational acceleration threshold (m/s²) |
| 0x02 | Throw-start rotational velocity threshold (°/s) |
| 0x03 | Impact acceleration threshold (m/s²) |
| 0x04 | Gravitational band minimum (m/s²) |
| 0x05 | Gravitational band maximum (m/s²) |

### 10.3 BLE UUIDs (Placeholder)

These UUIDs are placeholders. Use a UUID generator (e.g., `uuidgen`) to assign production UUIDs before Phase 1 implementation.

| Name | UUID |
|------|------|
| Kubb-Checker Service | `F4B20000-3E1A-4B7C-9A8D-2C6E1F0A5B3D` |
| Event Characteristic | `F4B20001-3E1A-4B7C-9A8D-2C6E1F0A5B3D` |
| Telemetry Characteristic | `F4B20002-3E1A-4B7C-9A8D-2C6E1F0A5B3D` |
| Status Characteristic | `F4B20003-3E1A-4B7C-9A8D-2C6E1F0A5B3D` |
| Control Characteristic | `F4B20004-3E1A-4B7C-9A8D-2C6E1F0A5B3D` |

### 10.4 Throw Detection State Machine

```
[REST] ──(accel > thr_throw_accel AND rot > thr_throw_rot)──► [PRE-THROW]
[PRE-THROW] ──(accel drops into gravity band 8.0–10.5 m/s²)──► [IN-FLIGHT]
[PRE-THROW] ──(threshold not sustained > 200 ms)──► [REST]   (false trigger)
[IN-FLIGHT] ──(accel spike > thr_impact)──► [POST-IMPACT] → compute + transmit event
[IN-FLIGHT] ──(timeout: flight > 5 s)──► [REST]             (missed impact)
[POST-IMPACT] ──(accel < rest_threshold for 500 ms)──► [REST]
[ANY STATE] ──(high-freq vibration burst < 500 ms, distinct from throw)──► [SHAKING]
[SHAKING] ──(vibration ceases for 500 ms)──► [REST]
```

### 10.5 BMI160 Key Registers

| Register | Address | Description |
|----------|---------|-------------|
| CHIP_ID | 0x00 | Expected value: 0xD1 |
| GYR_X_L/H | 0x0C/0x0D | Gyroscope X raw (16-bit signed) |
| GYR_Y_L/H | 0x0E/0x0F | Gyroscope Y raw |
| GYR_Z_L/H | 0x10/0x11 | Gyroscope Z raw |
| ACC_X_L/H | 0x12/0x13 | Accelerometer X raw (16-bit signed) |
| ACC_Y_L/H | 0x14/0x15 | Accelerometer Y raw |
| ACC_Z_L/H | 0x16/0x17 | Accelerometer Z raw |
| ACC_CONF | 0x40 | Accel ODR and bandwidth |
| ACC_RANGE | 0x41 | Accel full-scale range (0x0C = ±16 g) |
| GYR_CONF | 0x42 | Gyro ODR and bandwidth |
| GYR_RANGE | 0x43 | Gyro full-scale range (0x00 = ±2000 °/s) |
| CMD | 0x7E | Command register (0xB6 = soft reset) |

Full register map: `Documentation/bst-bmi160-ds000.pdf`.

### 10.6 LiIon 1S SoC Voltage Lookup Table

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

### 10.7 NTAG215 NFC Tag OOB Pairing Format

The NFC tag shall contain a BLE OOB pairing NDEF record. Minimum required fields:
- BLE Device Address (6 bytes, type `0x1B`)
- BLE Role: Peripheral (`0x01`)
- Security Manager OOB data (if required by NimBLE security configuration)

Recommended writing tool: NFC Tools (Android) or nRF Connect NFC tab → Write OOB BLE record.
