# application-machine-telemetry-accelerator

**Version:** 0.0.8
**Platform:** Fuuz ≥ 2026.2.0
**Publisher:** fuuz
**Enterprise:** MFGx
**Spec Version:** 2.0.0

---

## Overview

The Machine Telemetry Application is a comprehensive IoT data collection and analysis platform for manufacturing equipment. It extends the machine monitoring layer with deep sensor-level telemetry: capturing high-frequency machine data streams from OPC-UA devices and PLCs, storing both real-time current values and long-term historical time-series, detecting anomalies and threshold violations, generating alarms from telemetry conditions, and presenting trend analysis and process monitoring dashboards.

While `application-machine-monitoring-accelerator` focuses on operator-driven production tracking (OEE, mode changes, production counts), the Machine Telemetry Application focuses on the raw physics of the machine — pressures, temperatures, speeds, vibration, electrical values — and the automated detection logic built on top of those signals. It is suitable for predictive maintenance programs, process quality monitoring, SPC (Statistical Process Control), and OEE-linked condition monitoring.

---

## Package Contents

```
machine-telemetry/
├── manifest.json
├── definition.json
├── package-data.json
├── data/                        74 seed data files
├── dataFlows/                   40 automation flows
├── dataModels/                  48 data model definitions
├── savedTransforms/             18 reusable transforms
└── screens/                     26 UI screen definitions
```

---

## Functional Areas

### 1. Telemetry Data Collection

Continuous high-frequency data collection from OPC-UA device tags and PLC registers:

- **Device subscriptions** — Each monitored machine parameter is defined as an `IotTag` with OPC-UA node configuration; subscriptions are established via the platform's device gateway
- **Current value persistence** — `IotTag.currentValue` is updated on every subscription notification; `currentValueRecordedAt` tracks the timestamp
- **Historical storage** — Tags with `storeHistory = true` append every reading to a time-series collection (`IotTagHistoricalValue`), enabling trend analysis and replay
- **Tag use case routing** — `IotTagUseCase` (e.g., "Temperature", "Pressure", "Speed", "Vibration", "Counter", "Binary Signal") determines downstream processing: alarm evaluation, SPC calculation, OEE counter update, etc.

### 2. Alarm & Threshold Management

Configurable limit-based alarming on live sensor readings:

- **Alarm model** — `Alarm` records include `code` (auto-sequenced), `limitValue` (the threshold crossed), `deviation` (distance from setpoint at trigger time), timestamps for trigger/acknowledge/clear lifecycle
- **Alarm rules** — Per-tag or per-tag-type threshold configurations; supports high/low limits, rate-of-change limits, and deviation from setpoint
- **Alarm lifecycle** — new → acknowledged → cleared workflow; escalation logic with configurable time-based level-1/level-2 notification
- **Alarm-to-workcenter binding** — Alarms are linked to the workcenter associated with the triggering tag, enabling plant floor visibility integration
- **120-day data change capture** (exposed) — alarm state changes streamed for audit and SCADA integration

### 3. Process Monitoring & SPC

Statistical process control and process condition monitoring:

- Configurable control limits (UCL, LCL, UWL, LWL) per tag/parameter
- Rule violation detection (Western Electric rules, Nelson rules, or custom)
- Out-of-control signal logging with timestamps and deviation values
- Historical SPC chart data served to trend screens via integration flows

### 4. Trend & Analytics Dashboards

Time-series visualization for machine health and process performance:

- **Tag trend viewer** — Single-tag historical value chart with zoom, time range selection, and annotation support
- **Multi-tag comparison** — Overlay multiple telemetry parameters on a single time axis for correlation analysis
- **Machine health overview** — Per-workcenter dashboard showing current sensor values, alarm status, and recent trend sparklines
- **Alarm history timeline** — Chronological view of alarm events with deviation magnitudes and duration

### 5. IoT Device Configuration

Full device and tag management interface:

- Device registration with OPC-UA endpoint configuration
- Tag browser — import and configure OPC-UA node IDs from live device tree
- `WorkcenterIotTag` binding management — assign tags to workcenters for alarm routing and OEE counter integration
- Subscription health monitoring — view subscription status, last-update timestamps, and connection errors per device

### 6. Predictive Maintenance Support

Infrastructure for condition-based and predictive maintenance programs:

- Tag-level baseline configuration for deviation alerting
- Historical trend data retention for training predictive models
- Alarm pattern analysis screens for identifying recurring failure modes
- Integration hooks for exporting tag history to external ML/analytics systems

---

## Data Models (48 total)

### Telemetry Core

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Alarm` | code (auto-seq), limitValue!, deviation, occurAt, acknowledgedAt, clearedAt, workcenterId, iotTagId, alarmRuleId | Threshold violation event; 120-day data capture |
| `IotTag` | name!, active!, storeHistory!, currentValue (JSON), currentValueRecordedAt, configuration (JSON), iotTagTypeId, iotTagUseCaseId, deviceId | OPC-UA tag subscription record |
| `IotTagHistoricalValue` | recordedAt!, value (JSON!), iotTagId | Append-only time-series log of tag readings |
| `IotTagType` | name! (=id auto), usable!, configurationSchema | Tag type classification (e.g., Analog, Digital, Counter) |
| `IotTagUseCase` | name! (=id auto), usable!, configurationSchema | Processing routing classification |

### Alarm Configuration

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `AlarmRule` | name!, highLimit, lowLimit, highHighLimit, lowLowLimit, deadband, enabled, iotTagId, iotTagTypeId | Configurable threshold rules per tag or tag type |
| `AlarmStatus` | name!, active, new, acknowledged, cleared, canceled, usable! | Alarm lifecycle state definitions |
| `AlarmEscalation` | levelOneMinutes, levelTwoMinutes, levelOneRoleId, levelTwoRoleId, alarmStatusId | Escalation timer and routing configuration |

### Device & Connectivity

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Device` | name!, active!, connectionStatus, lastConnectedAt, endpoint, deviceTypeId, gatewayId | OPC-UA/PLC device registration |
| `DeviceType` | name!, protocol (OPC-UA, Modbus, etc.), configurationSchema | Device protocol classification |
| `DeviceGateway` | name!, active!, host, port, status | Edge gateway for OPC-UA connectivity |
| `WorkcenterIotTag` | workcenterId, iotTagId | Many-to-many binding of tags to workcenters |

### Process Monitoring

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `SpcConfiguration` | name!, ucl, lcl, uwl, lwl, iotTagId, ruleSetId | Statistical process control limits per tag |
| `SpcSample` | value!, recordedAt!, spcConfigurationId, violationFlags (JSON) | Individual SPC sample with rule check results |
| `ControlChart` | name!, type (Xbar-R, Xbar-S, I-MR), subgroupSize, iotTagId | Control chart configuration |

### Reference Data

| Model | Key Fields | Purpose |
|-------|-----------|---------|
| `Workcenter` | code!, name!, active! | Machine entity linkage (cross-reference) |
| `Facility` | code!, name!, type | Site hierarchy node |
| Additional models (33) | Various | Supporting tag metadata, calibration records, maintenance schedules, notification templates, and dashboard configurations |

---

## Data Flows (40 total)

### Core Telemetry Pipeline

| Flow | Type | Description |
|------|------|-------------|
| IOT Tag Handler | System | Routes incoming device subscription updates by use case to production/alarm/OEE handlers |
| Update Iot Tag by Device Subscriptions | System | Processes subscription events, updates `IotTag.currentValue` and `currentValueRecordedAt` |
| Create Historical IOT Tag Values | System | Appends tag readings to `IotTagHistoricalValue` when `storeHistory = true` |
| Create Device Subscription From Iot Tags | System | Generates OPC-UA subscriptions on the device gateway from IotTag configurations |

### Alarm Processing

| Flow | Type | Description |
|------|------|-------------|
| Evaluate Alarm Rules | System | Triggered on each tag update; checks configured `AlarmRule` thresholds and creates `Alarm` records on violation |
| Alarm Escalation Timer | System | Periodic check; fires level-1/level-2 notifications when open alarms exceed configured time thresholds |
| Alarm State Machine | System | Processes alarm lifecycle transitions (acknowledge, clear, cancel) |
| Alarm Notification Dispatch | System | Routes alarm notifications to assigned roles/users via configured channels |

### Analytics & Reporting

| Flow | Type | Description |
|------|------|-------------|
| Tag Trend Query | Integration | Returns paginated historical tag values for a given time range and tag ID; used by trend screens |
| SPC Evaluation | System | Evaluates control chart rules on new samples; logs violations to `SpcSample.violationFlags` |
| Machine Health Aggregator | System | Computes per-workcenter health scores from active alarm count, tag deviation metrics, and uptime |
| Export Tag History | Integration | Bulk export of historical tag values to CSV or JSON for external analytics |

### Device Management

| Flow | Type | Description |
|------|------|-------------|
| Device Connectivity Check | System | Periodic ping of registered device gateways; updates `Device.connectionStatus` |
| Tag Discovery | Integration | Browses OPC-UA node tree on a device and returns available tag paths for configuration |
| Subscription Health Monitor | System | Detects stale subscriptions (tags not updated beyond threshold) and triggers reconnection |

*Additional flows (25+) cover SPC charting, notification templates, Fuuz-to-SCADA publishing, trend data caching, and dashboard data preparation.*

---

## Screens (26 total)

**Operator / Plant Floor**
- Machine health overview per workcenter — current sensor readings, alarm count, mode/OEE summary
- Active alarm list with acknowledge/clear actions
- Tag current value monitor — live readout of all tags for a selected workcenter

**Trend & Analytics**
- Single-tag trend chart with configurable time range and resolution
- Multi-tag overlay comparison dashboard
- SPC control chart viewer with violation highlighting
- Alarm history timeline with filter by workcenter, tag, alarm type, and date range

**Configuration**
- Device management — register, edit, test connectivity
- Tag configuration — create IotTags, set use case, configure alarm rules, enable history
- WorkcenterIotTag binding manager
- Alarm rule configuration — set high/low limits, deadband, escalation timers
- SPC configuration — define control limits, select rule sets
- Alarm status and escalation configuration

---

## Seed Data (74 records)

Comprehensive reference data:

- **IotTagTypes** — Analog Input, Digital Input, Counter, Accumulator, Calculated, String, and protocol-specific types
- **IotTagUseCases** — Temperature, Pressure, Speed, Vibration, Power, Current, Voltage, Flow Rate, Counter, Binary Signal, Setpoint Deviation, Generic, and additional process-specific use cases
- **AlarmStatuses** — new, active, acknowledged, cleared, canceled with Boolean flag presets
- **DeviceTypes** — OPC-UA, Modbus TCP, Modbus RTU, EtherNet/IP, BACnet, MQTT, and generic HTTP types
- **Default SPC rule sets** — Western Electric, Nelson, AIAG, and custom rule definitions
- **Default control chart templates** — Xbar-R, Xbar-S, I-MR pre-configured templates
- **Notification templates** — Email and push notification templates for alarm level-1 and level-2 escalations
- **Dashboard layout configurations** — Default screen layouts for plant floor and management views

---

## Installation

1. Ensure Fuuz platform version ≥ 2026.2.0 is deployed
2. Import via Fuuz Package Manager — seed data (74 records) applies automatically
3. Register device gateways and OPC-UA devices via the Device Management screen
4. Use Tag Discovery to browse OPC-UA nodes and create `IotTag` records for each monitored parameter
5. Assign tags to workcenters via the WorkcenterIotTag Binding Manager
6. Configure `IotTagUseCase` on each tag to enable appropriate downstream processing
7. Set up `AlarmRule` threshold configurations per tag or tag type
8. Enable `storeHistory = true` on tags requiring trend analysis
9. Run *Create Device Subscription From Iot Tags* flow to activate live data collection
10. Configure SPC control limits and chart types for process monitoring tags
11. Set up escalation rules and notification assignments for alarm routing

---

## Dependencies

- **Fuuz Platform** ≥ 2026.2.0
- **Device Gateway** — required for OPC-UA/Modbus connectivity (Fuuz Edge Gateway or cloud gateway)
- **`application-machine-monitoring-accelerator`** (optional) — integrates OEE counter updates and workcenter state linkage
- **Notification module** — required for alarm escalation level-1/level-2 delivery
- **Scheduler module** — required for periodic alarm escalation checks and device connectivity monitoring

---

*Built on the [Fuuz Industrial Operations Platform](https://fuuz.com)*

## Service levels

No service level agreement applies to anything published here. It becomes a supported
deliverable only once it has been implemented by a Fuuz services professional or an
approved Fuuz partner.
