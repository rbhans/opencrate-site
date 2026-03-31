# Operations

## Discovery

Discovery finds devices on the network and brings them into OpenCrate. The system is protocol-agnostic -- the same workflow applies regardless of whether a device speaks BACnet or Modbus.

### How it works

Every discovered device gets a `DiscoveredDevice` record with an id, protocol identifier, connection status (Online / Offline / Unknown), vendor and model strings, a network ID, and a `protocol_meta` JSON blob that holds protocol-specific details. Each device contains `DiscoveredPoint` entries that capture the protocol binding, point kind (Analog / Binary / Multistate), writability, and any state labels.

Devices move through three states:

1. **Discovered** -- The device was found on the network but no action has been taken.
2. **Accepted** -- An operator reviewed the device and brought it into the system. Accepting a device creates nodes, entities, and auto-tags in one pass.
3. **Ignored** -- The operator chose to exclude this device.

Scans are always user-initiated. OpenCrate never discovers devices automatically on startup. When a scan runs again later, previously accepted devices stay accepted -- re-discovery preserves state.

### BACnet scanning

A BACnet scan broadcasts Who-Is, walks each responding device's object list, merges results into the bridge, and upserts records to the discovery store. The scan enriches devices with vendor name, model name, firmware revision, location, description, and other metadata from the device's standard properties.

### Modbus scanning

Modbus discovery works in three modes depending on the configuration:

- **Configured devices** -- Reads devices already defined in the scenario, then enriches each with FC43 (Read Device Identification) to pull vendor, product, and revision strings.
- **TCP scan** -- Probes a range of unit IDs at a given host and port. Each responder gets an FC43 query for identification.
- **RTU scan** -- Same unit ID probe, but over the serial bus.

Because Modbus devices do not self-describe their register maps, discovery points operators to the profile library and the Registers tab for manual or profile-based point configuration.

### Auto-naming and grouping

The `naming.rs` module applies heuristics to generate human-readable names from protocol metadata. The `grouping.rs` module analyzes discovered points and suggests equipment groupings, making it faster to organize large sites.

### Atlas integration

When the `atlas` feature gate is enabled, discovered points are automatically tagged using a semantic haystack matcher. This is optional and not part of the default build.

---

## Commissioning

Once devices are accepted, the commissioning workflow verifies that every point works correctly before the system goes live.

### Sessions and items

Each accepted device gets a commissioning session. Sessions progress through a fixed lifecycle:

**not_started** --> **in_progress** --> **completed** --> **signed_off**

Within each session, individual points get checklist items. The system auto-generates items based on point capabilities:

| Item type | Created when |
|---|---|
| ReadVerify | Always -- every point needs a read check |
| WriteVerify | Point is writable |
| AlarmVerify | Point is alarmable |
| ScheduleVerify | Point is schedulable |

Each item moves independently through **not_started**, **verified**, **failed**, or **deferred**. As items are completed, the session status auto-promotes. The commissioning overview dashboard in the GUI shows progress across all devices.

### Storage

The `CommissioningStore` uses the same mpsc + SQLite thread pattern as other stores, persisting to `data/commissioning.db` with tables for sessions and items.

---

## Alarms

### Lifecycle

Every alarm follows a two-flag lifecycle:

```
raised
  |
  +--> cleared (condition resolved, not yet acknowledged)
  +--> acknowledged (operator saw it, condition may still be active)
  |
  +--> cleared + acknowledged (fully resolved)
```

An alarm is considered fully resolved only when both flags are set. The `AlarmStore` maintains active alarms and a full history including acknowledgment events.

### Flag synchronization

Alarm flags are synchronized through the EventBus. When a value changes, the alarm evaluator checks conditions and publishes `AlarmRaised` or `AlarmCleared` events immediately. A periodic stale check runs every 30 seconds to catch any conditions that the event-driven path might miss.

### Shelving

Operators can shelve alarms to suppress them temporarily. Each shelving action includes an expiry time. A cleanup tick runs every 5 minutes to automatically unshelve expired entries. Shelving records are stored in the notification store alongside routing rules.

### GUI

The alarm view shows active alarms with Ack, Ack All, and Shelve buttons. A separate history view provides the full alarm record with export capability.

---

## Alarm routing and notifications

### Notification channels

OpenCrate dispatches alarm notifications through three channel types:

- **Webhook** -- HTTP POST with configurable URL and headers (via reqwest).
- **Email** -- SMTP or HTTP API delivery (via lettre).
- **SMS** -- Twilio integration or generic HTTP POST.

### Routing rules

The `AlarmRouter` subscribes to the EventBus and reacts to `AlarmRaised`, `AlarmCleared`, and `AlarmAcknowledged` events. Routing rules filter by severity, device, and alarm type. Rules support tiered escalation -- if a first-tier recipient does not act, the alert escalates to the next tier.

Failed deliveries retry with exponential backoff: 30-second base, 1-hour maximum, up to 8 attempts. The notification log records every delivery attempt with its outcome. A badge in the GUI toolbar shows the count of failed notifications.

### Webhook subscriptions

For more granular outbound integration, the webhook system provides per-endpoint configuration with event toggles for alarm raised/cleared/acknowledged and device down/recovered. Each endpoint can filter by severity and tags (AND logic via NodeStore lookup).

Provider-specific formatters shape payloads for different targets:

| Provider | Format |
|---|---|
| Generic | JSON body with HMAC-SHA256 signature |
| Slack | Block Kit message |
| Teams | Adaptive Card |
| PagerDuty | Events API v2 |
| ntfy | Priority headers |

Deliveries retry at 5s, 30s, and 120s intervals. A global pause switch stops all dispatching. The delivery log tracks every attempt with status filtering in the GUI.

### MQTT publishing

The MQTT publisher subscribes to the EventBus and forwards events to one or more MQTT brokers. Published event types include `ValueChanged`, `AlarmRaised`, `AlarmCleared`, `DeviceDown`, and `DeviceDiscovered`. Payloads are JSON.

Topic patterns use `{variable}` templates with automatic sanitization:

- Point values: `opencrate/{device_id}/{point_id}/value`
- Alarms: `opencrate/alarms/{severity}/{node_id}`
- Device status: `opencrate/status/{device_key}`

Each broker runs its own event loop task with auto-reconnect. Configuration changes are picked up via a watch channel for hot-reload without restart.

### Storage

- **NotificationStore** -- `data/notifications.db` with tables for recipients, routing rules, alarm shelving, and the notification log.
- **WebhookStore** -- `data/webhooks.db` with tables for endpoints, deliveries, and config.
- **MqttStore** -- `data/mqtt.db` with tables for brokers and topic patterns.

---

## Schedules

### Weekly and exception schedules

The `ScheduleStore` manages three entities:

- **Weekly schedules** -- Define time blocks for each day of the week.
- **Exception schedules** -- Override weekly schedules for holidays, special events, or maintenance windows.
- **Assignments** -- Bind schedules to points so that values are written automatically.

### BACnet schedule sync

For BACnet devices, OpenCrate can read and write schedules directly from the device. The sync panel in the schedule editor provides Read from Device and Write to Device buttons that call:

- `read_schedule` / `write_schedule` -- The weekly schedule object.
- `read_calendar` -- Calendar entries that exception schedules reference.
- `read_exception_schedule` -- The exception schedule entries themselves.
- `read_schedule_default` -- The default value when no schedule entry is active.

This two-way sync keeps the BAS database and the field device in agreement.

---

## History

### Collection modes

The `HistoryStore` records time-series samples in two modes:

- **Interval** -- Samples taken at a fixed period regardless of whether the value changed.
- **COV (Change of Value)** -- A sample is recorded only when the value changes. The Modbus poll loop uses `set_if_changed()` to skip duplicate writes.

### Queries and export

History can be queried by time range through the API at `/api/history`. Results can be exported to CSV for offline analysis or imported into external tools.

---

## Auth and audit

### Authentication

Passwords are hashed with Argon2 using a random salt per user. Authentication issues JWT tokens that the client includes on subsequent requests.

On first run, `POST /api/auth/setup` creates the initial admin account. No default credentials exist.

### Roles and permissions

Three roles provide coarse access control:

| Role | Access level |
|---|---|
| Admin | Full access to all features |
| Operator | Day-to-day operations and configuration |
| Viewer | Read-only access |

Within those roles, 16 granular permissions control specific actions:

| Permission | Scope |
|---|---|
| WritePoints | Write values to field devices |
| AcknowledgeAlarms | Acknowledge active alarms |
| ManageSchedules | Create and edit schedules |
| ManageDiscovery | Run scans, accept/ignore devices |
| ManagePrograms | Create and edit logic programs |
| ManageVirtualPoints | Create and edit virtual points |
| ManageUsers | User account administration |
| ManageNotifications | Configure notification channels and rules |
| ManageMqtt | Configure MQTT brokers and topics |
| ManageCommissioning | Run commissioning workflows |
| ManageReports | Create and schedule reports |
| ManageEnergy | Configure meters, rates, and baselines |
| ManageWebhooks | Configure webhook endpoints |
| ManageFdd | Configure FDD rules and bindings |
| ManageExport | Configure export connectors |
| ViewAudit | View the audit log |

### Audit trail

The `AuditStore` logs every operator action with a timestamp, the acting user, the action taken, and a details blob. The audit log is accessible through the GUI for users with the `ViewAudit` permission.

---

## Weather

### Service architecture

The `WeatherService` uses pluggable adapters to fetch weather data. Two providers ship by default:

- **OpenWeatherMap** -- Uses the OWM API with an API key.
- **Custom HTTP** -- Calls any HTTP endpoint that returns the expected JSON shape.

### Data models

Weather data is structured as `WeatherForecast` containing `HourlyForecast` and `DailyForecast` entries with temperature, humidity, wind, and condition fields.

The service supports auto-geocoding from a building name or address, so operators do not need to manually enter coordinates. Weather data appears in a dedicated view and as a widget in the toolbar.

---

## Backup

### Scheduled backups

The `BackupScheduler` runs on a configurable interval (specified in hours) and saves database files to a standard backup location. A retention count controls how many backups are kept -- older backups are automatically cleaned up.

### API

Three operations are available at `/api/system/backup`:

- **Trigger** -- Start a backup immediately.
- **List** -- Show available backups.
- **Config** -- Read or update the backup schedule and retention settings.

Configuration is persisted to `data/backup_config.json`.
