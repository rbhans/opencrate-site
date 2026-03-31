# Outbound Connectors

All outbound connectors are EventBus subscribers. They start during platform initialization, run for the lifetime of the process, and operate independently of each other and of the field protocol drivers.

---

## MQTT

The MQTT integration streams real-time telemetry, alarms, and device status to one or more MQTT brokers. It uses the `rumqttc` 0.24 client library.

### Architecture

`MqttPublisher` subscribes to the EventBus (following the same pattern as the AlarmRouter) and publishes JSON payloads for the following event types:

| Event | Default topic pattern |
|-------|----------------------|
| `ValueChanged` | `opencrate/{device_id}/{point_id}/value` |
| `AlarmRaised` | `opencrate/alarms/{severity}/{node_id}` |
| `AlarmCleared` | `opencrate/alarms/{severity}/{node_id}` |
| `DeviceDown` | `opencrate/status/{device_key}` |
| `DeviceDiscovered` | `opencrate/status/{device_key}` |

Each broker runs its own event loop task with automatic reconnection. Configuration changes are picked up through a watch channel for hot-reload without restarting the process.

### Topic engine

Topics use `{variable}` template syntax. Variables are extracted from the event context and sanitized (special characters replaced, empty segments removed) before interpolation. The defaults above can be overridden per topic pattern.

### Storage

`MqttStore` uses the standard mpsc + SQLite thread pattern with a database at `data/mqtt.db` containing two tables:

| Table | Purpose |
|-------|---------|
| `mqtt_broker` | Broker connection configurations (host, port, credentials, TLS settings) |
| `mqtt_topic_pattern` | Topic templates with event type mappings |

### Configuration

Brokers and topic patterns are managed through the GUI (Config > MQTT tab with Brokers, Topics, and Status sub-tabs) or directly in the SQLite database. The MQTT tab is gated by the `ManageMqtt` permission.

---

## Webhooks

The webhook system delivers HTTP POST notifications for alarm and device events to external services, with built-in support for formatting payloads for popular platforms.

### Architecture

`WebhookDispatcher` subscribes to the EventBus and fires HTTP POST requests for:

- `AlarmRaised`, `AlarmCleared`, `AlarmAcknowledged`
- `DeviceDown`, `DeviceRecovered`
- FDD (Fault Detection and Diagnostics) events

### Per-endpoint configuration

Each webhook endpoint has:

- **Event toggles:** Enable or disable specific event types per endpoint.
- **Severity filter:** Only forward alarms at or above a minimum severity level.
- **Tag filter:** Only forward events for nodes matching all specified tags (AND logic). Tag resolution goes through the NodeStore.
- **Provider format:** Select the payload format (see below).

### Provider formatters

| Provider | Format | Authentication |
|----------|--------|----------------|
| **Generic** | JSON payload | HMAC-SHA256 signature in `X-OpenCrate-Signature` header |
| **Slack** | Block Kit message format | Webhook URL |
| **Teams** | Adaptive Cards | Webhook URL |
| **PagerDuty** | Events API v2 | Routing key |
| **ntfy** | Plain text with priority headers | Topic-based (URL path) |

The Generic provider includes alarm enrichment -- the payload contains full node context, alarm details, and timestamps.

### Retry policy

Failed deliveries are retried on an escalating schedule:

1. First retry after **5 seconds**
2. Second retry after **30 seconds**
3. Third retry after **120 seconds**

After all retries are exhausted, the delivery is marked as permanently failed in the delivery log.

### Global pause

The dispatcher supports a global pause toggle that suspends all webhook deliveries. Events that arrive during a pause are not queued -- they are dropped. This is intended for maintenance windows.

### Storage

`WebhookStore` uses `data/webhooks.db` with three tables:

| Table | Purpose |
|-------|---------|
| `webhook_endpoint` | Endpoint URLs, provider type, event toggles, filters |
| `webhook_delivery` | Delivery log with status, attempt count, response codes |
| `webhook_config` | Global configuration (pause state) |

### API

Eight REST endpoints under `/api/webhooks/`:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/webhooks` | GET | List all endpoints |
| `/api/webhooks` | POST | Create endpoint |
| `/api/webhooks/{id}` | GET | Get endpoint details |
| `/api/webhooks/{id}` | PUT | Update endpoint |
| `/api/webhooks/{id}` | DELETE | Delete endpoint |
| `/api/webhooks/{id}/test` | POST | Send a test delivery |
| `/api/webhooks/deliveries` | GET | Query delivery log (supports status filter) |
| `/api/webhooks/config` | GET/PUT | Read or update global config |

All webhook endpoints require the `ManageWebhooks` permission (Admin-only). Mutations are recorded in the audit log with actions: CreateWebhook, UpdateWebhook, DeleteWebhook, TestWebhook.

### GUI

The Config > Webhooks tab provides:

- **Endpoints list:** CRUD operations and test delivery button for each endpoint.
- **Delivery log:** Filterable by status (success, failed, pending).

---

## Export

The export system replicates OpenCrate data to external time-series databases and analytics platforms for long-term storage and visualization.

### Architecture

The export system is built around the `ExportConnector` trait, which defines the interface for writing batches of history and alarm data to external systems. `ExportPublisher` subscribes to the EventBus and routes events to configured connectors.

### Connectors

**InfluxDB**

Writes data via the InfluxDB HTTP API:

- `write_history_batch` -- Writes point history as InfluxDB line protocol.
- `write_alarm_batch` -- Writes alarm events as InfluxDB line protocol.

Always available in the default build.

**PostgreSQL**

Feature-gated behind `export-postgres`. Writes to PostgreSQL tables using a connection pool. Enable with:

```
cargo build --features export-postgres
```

### Per-connector event toggles

Each configured export connector can independently enable or disable forwarding of specific event types (value changes, alarms, device status), allowing fine-grained control over what data leaves the system.

### Backfill

`ExportPublisher` supports replaying historical data from a specified start time. This is useful for:

- Initial population of a new external database
- Recovering from export outages
- Migrating between analytics platforms

Backfill reads from the local history store and feeds records through the normal export pipeline.

### Storage

`ExportStore` uses `data/export.db` to track connector configurations and export state (last exported timestamp, backfill progress).

### API

Ten REST endpoints under `/api/energy/` cover meter CRUD, rate CRUD, summary, consumption queries, and CSV export. Export configuration is managed through the Config > Energy tab in the GUI.

The `ManageEnergy` permission (Admin and Operator roles) controls access, with eight audit action types for tracking changes.
