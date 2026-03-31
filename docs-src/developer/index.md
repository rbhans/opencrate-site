# Developer Guide

## Codebase map

The `src/` directory contains approximately 25 modules. Here is what each one does and where to find it.

### Core infrastructure

| Module | Path | Description |
|--------|------|-------------|
| **event bus** | `src/event/bus.rs` | Broadcast-based event bus (capacity 4096). All events are wrapped in `Arc<Event>` to avoid cloning. This is the central nervous system -- every store, engine, and connector communicates through it. |
| **node model** | `src/node/mod.rs` | The unified object model. A `Node` has an ID, type (Site/Space/Equip/Point/VirtualPoint), display name, parent, value, status, tags, refs, properties, capabilities, and protocol binding. |
| **platform init** | `src/platform.rs` | `init_platform(scenario_path, profiles_dir)` creates the EventBus, all stores, starts all engines and bridges, and returns a `Platform` struct. Shared between CLI and GUI entry points. |

### Stores (data layer)

Every store follows the same pattern: an mpsc channel sends commands to a dedicated thread that owns a SQLite connection. The public API is a set of async methods that send commands and await responses through oneshot channels. This keeps SQLite access single-threaded while exposing an async interface to the rest of the system.

| Store | Database | Tables | Description |
|-------|----------|--------|-------------|
| `NodeStore` | `data/nodes.db` | 6 (node, node_tag, node_ref, node_property, node_capability, protocol_binding) | Two-layer: hot cache (`Arc<RwLock<HashMap>>`) + SQLite. Full CRUD, tag/ref/property ops, hierarchy queries via recursive CTE. |
| `PointStore` | `data/points.db` | -- | Thread-safe point value store. Publishes ValueChanged to EventBus. `set_if_changed()` for COV filtering. |
| `EntityStore` | `data/entities.db` | -- | SQLite entity store. Publishes EntityCreated/Updated/Deleted to EventBus. |
| `AlarmStore` | `data/alarms.db` | -- | Active and historical alarms. EventBus-driven flag sync (replaces old 3-sec poll loop), periodic stale check every 30s. |
| `HistoryStore` | `data/history.db` | -- | Time-series point history. |
| `ScheduleStore` | `data/schedules.db` | -- | Weekly schedules, exception schedules, calendar entries. |
| `DiscoveryStore` | `data/discovery.db` | -- | Discovered devices and points. Tracks device state (Discovered/Accepted/Ignored), connection status, network ID. |
| `ProgramStore` | `data/programs.db` | -- | Logic engine programs, execution logs (auto-pruned to last 100 per program). `subscribe()` for version change notifications. |
| `NotificationStore` | `data/notifications.db` | 4 (recipient, routing_rule, alarm_shelving, notification_log) | Alarm routing configuration. |
| `MqttStore` | `data/mqtt.db` | 2 (mqtt_broker, mqtt_topic_pattern) | MQTT broker and topic config. Watch channel for hot-reload. |
| `WebhookStore` | `data/webhooks.db` | 3 (webhook_endpoint, webhook_delivery, webhook_config) | Webhook endpoints, delivery log, global config. |
| `CommissioningStore` | `data/commissioning.db` | 2 (commission_session, commission_item) | Per-device commissioning sessions and checklists. |
| `ReportStore` | `data/reports.db` | 3 (report_definition, report_schedule, report_execution) | Report definitions, schedules, execution history. |
| `EnergyStore` | `data/energy.db` | 4 (utility_rate, energy_meter, energy_baseline, energy_rollup) | Energy analytics data. Upsert on conflict for partial/final rollups. |
| `ExportStore` | `data/export.db` | -- | Export connector configurations and state. |
| `AuditStore` | -- | -- | Audit log entries. |

### Protocol drivers

| Module | Path | Description |
|--------|------|-------------|
| **protocol traits** | `src/protocol/` | `ProtocolDriver` trait (replaces PointSource for new code), `ValueSink` trait, `RawProtocolValue` struct, `ProfileNormalizer` (dispatches by protocol string). |
| **BACnet bridge** | `src/bridge/bacnet.rs` | Full BACnet client with all service support. See [Field Protocols](../integrations/field-protocols.md). |
| **Modbus bridge** | `src/bridge/modbus.rs` | Modbus TCP/RTU client with block reads and full FC coverage. |
| **bridge traits** | `src/bridge/traits.rs` | `PointSource` trait (still used by existing bridges). |
| **backoff** | `src/bridge/backoff.rs` | Shared `DeviceBackoff` for exponential reconnection (used by both BACnet and Modbus). |

### Engines and automation

| Module | Path | Description |
|--------|------|-------------|
| **logic engine** | `src/logic/` | Visual block programming compiled to Rhai scripts. `ExecutionEngine` runs programs on a 1-second tick with EventBus-triggered on-change execution. |
| **alarm router** | `src/notification/router.rs` | EventBus subscriber for alarm events. Rule-based routing to notification channels with tiered escalation and exponential retry. |
| **MQTT publisher** | `src/mqtt/publisher.rs` | EventBus subscriber. Per-broker event loop with auto-reconnect and hot-reload. |
| **webhook dispatcher** | `src/webhook/dispatcher.rs` | EventBus subscriber. HTTP POST with retry, per-endpoint filtering, provider formatting. |
| **reporting engine** | `src/reporting/engine.rs` | Queries multiple stores to generate report sections. 7 section types. |
| **report renderer** | `src/reporting/renderer.rs` | Inline-CSS HTML rendering (email-safe, dark theme, no external dependencies). |
| **report scheduler** | `src/reporting/` | 60-second tick loop, SMTP delivery via lettre. |
| **energy analytics** | `src/energy/` | 6 sub-modules: consumption, demand, degree_days, cost, rollup, scheduler. 15-minute background calculation cycle. |
| **FDD** | `src/fdd/` | Fault Detection and Diagnostics engine. |
| **export publisher** | `src/export/` | EventBus subscriber. Per-connector event routing with backfill support. |

### API layer

| Module | Path | Description |
|--------|------|-------------|
| **API root** | `src/api/mod.rs` | Axum router setup with `ApiState` (holds all stores and shared state). |
| **route modules** | `src/api/routes/` | Route handlers organized by domain (energy, export, fdd, webhooks, plus existing routes in `mod.rs`). |
| **auth** | `src/auth.rs` | JWT authentication, permission system (ManageNotifications, ManageMqtt, ManageWebhooks, ManageReports, ManageEnergy, ManageCommissioning, ShelveAlarm, etc.). |

### GUI layer

| Module | Path | Description |
|--------|------|-------------|
| **app entry** | `src/gui/app.rs` | Dioxus application root. |
| **state** | `src/gui/state.rs` | `AppState` struct holding all stores, event bus, BACnet/Modbus bridge handles. |
| **components** | `src/gui/components/` | 46+ Dioxus components including config_view, energy_view, fdd_view, export_settings, webhook_settings, and protocol-specific views. |

### Configuration

| Module | Path | Description |
|--------|------|-------------|
| **profiles** | `src/config/profile.rs` | `DeviceProfile` and `Point` definitions with protocol bindings. Loaded from JSON files in `profiles/`. |
| **scenarios** | `src/config/scenario.rs` | `ScenarioConfig` and `DeviceInstance`. References profiles and defines the site. |
| **loader** | `src/config/loader.rs` | `resolve_scenario()` loads scenario + profiles and resolves references. |
| **templates** | `src/config/template.rs` | `SystemTemplate` / `TemplateNode` for pre-built site hierarchies (Small Office, School, Hospital). `auto_create_nodes()` creates equipment and point nodes with auto-tagging. |

### Discovery

| Module | Path | Description |
|--------|------|-------------|
| **discovery service** | `src/discovery/` | Protocol-agnostic discovery orchestration. `DiscoveryService` manages scan, accept/ignore workflows. |
| **BACnet adapter** | `src/discovery/bacnet_adapter.rs` | Converts `BacnetDevice`/`BacnetObject` to protocol-agnostic `DiscoveredDevice`/`DiscoveredPoint`. |
| **BACnet units** | `src/discovery/bacnet_units.rs` | ASHRAE 135 Annex K unit mapping (u32 to display string, ~50 units). |

### Plugin system

| Module | Path | Description |
|--------|------|-------------|
| **plugin traits** | `src/plugin/mod.rs` | `ProtocolPlugin`, `ProtocolBridgeHandle` (object-safe with `as_any()` for downcast), `HistoryBackend`, `AlarmEvaluator`, `LogicEnginePlugin`, `ImportExportPlugin`. |
| **registry** | `src/plugin/mod.rs` | `PluginRegistry` with `find_protocol()`, `protocol_ids()`. No dynamic loading -- plugins are registered at startup. |

## The store pattern

Every store in OpenCrate follows the same architecture. Understanding this pattern is essential for contributing.

```
                    ┌──────────────┐
  async caller ───► │  mpsc sender │ ──► dedicated thread ──► SQLite
                    └──────────────┘          │
                           ▲                 │
                           │                 ▼
                    oneshot receiver ◄── oneshot sender
```

The public interface exposes async methods. Internally:

1. A command enum defines all operations (e.g., `StoreCmd::Get`, `StoreCmd::Insert`).
2. Each method creates a oneshot channel, wraps the request + sender half into a command, and sends it through the mpsc channel.
3. A dedicated thread receives commands, executes SQLite queries, and sends results back through the oneshot.
4. Stores accept `Option<EventBus>` in their constructor. Pass `None` in tests to avoid event wiring.

Sketch of a typical store:

```rust
enum Cmd {
    Get { id: String, reply: oneshot::Sender<Option<MyRecord>> },
    Insert { record: MyRecord, reply: oneshot::Sender<Result<()>> },
}

pub struct MyStore {
    tx: mpsc::Sender<Cmd>,
}

impl MyStore {
    pub fn new(db_path: &str, event_bus: Option<EventBus>) -> Self {
        let (tx, rx) = mpsc::channel(256);
        std::thread::spawn(move || {
            let conn = Connection::open(db_path).unwrap();
            // Run migrations
            while let Some(cmd) = rx.blocking_recv() {
                match cmd {
                    Cmd::Get { id, reply } => {
                        let result = /* query */;
                        let _ = reply.send(result);
                    }
                    Cmd::Insert { record, reply } => {
                        let result = /* insert */;
                        if let Some(bus) = &event_bus {
                            let _ = bus.send(Event::EntityCreated { /* ... */ });
                        }
                        let _ = reply.send(result);
                    }
                }
            }
        });
        Self { tx }
    }

    pub async fn get(&self, id: &str) -> Option<MyRecord> {
        let (reply_tx, reply_rx) = oneshot::channel();
        self.tx.send(Cmd::Get { id: id.to_string(), reply: reply_tx }).await.ok()?;
        reply_rx.await.ok()?
    }
}
```

## Event bus

The `EventBus` is a broadcast channel with capacity 4096. Events are wrapped in `Arc<Event>` to avoid cloning payloads across multiple subscribers.

### Subscribing

```rust
let mut rx = event_bus.subscribe();
tokio::spawn(async move {
    while let Ok(event) = rx.recv().await {
        match event.as_ref() {
            Event::ValueChanged { node_id, value, .. } => {
                // Handle value change
            }
            Event::AlarmRaised { .. } => {
                // Handle alarm
            }
            _ => {}
        }
    }
});
```

### Event variants

ValueChanged, StatusChanged, AlarmRaised, AlarmCleared, AlarmAcknowledged, ScheduleWritten, EntityCreated, EntityUpdated, EntityDeleted, DeviceDiscovered, DeviceDown, DeviceAccepted, DiscoveryScanComplete, DeviceMonitorCycle, ObjectListChanged.

## Testing

### Store tests

Pass `None` for the EventBus to avoid requiring event infrastructure:

```rust
#[tokio::test]
async fn test_my_store() {
    let store = MyStore::new(":memory:", None);
    store.insert(MyRecord { /* ... */ }).await.unwrap();
    let result = store.get("test-id").await;
    assert!(result.is_some());
}
```

Use `:memory:` for the database path in tests so each test gets a fresh database with no filesystem cleanup needed.

### Integration tests

For tests that need event bus interaction:

```rust
#[tokio::test]
async fn test_with_events() {
    let event_bus = EventBus::new();
    let mut rx = event_bus.subscribe();
    let store = MyStore::new(":memory:", Some(event_bus.clone()));

    store.insert(MyRecord { /* ... */ }).await.unwrap();

    let event = rx.recv().await.unwrap();
    assert!(matches!(event.as_ref(), Event::EntityCreated { .. }));
}
```

## Configuration formats

### Scenario file (`scenario.json`)

The scenario file defines the site: which devices are present, what profiles they use, and protocol-specific settings.

```json
{
  "name": "My Building",
  "devices": [
    {
      "id": "ahu-1",
      "profile": "ahu-35pt",
      "instance_id": 1001,
      "description": "Main AHU"
    }
  ],
  "settings": {
    "bacnet_networks": {
      "ip-main": {
        "mode": "ip",
        "monitor_interval_secs": 300
      }
    },
    "modbus": {
      "mode": "tcp",
      "default_timeout_ms": 1000
    }
  }
}
```

### Device profile (`profiles/*.json`)

Profiles describe a device type's points with their protocol bindings:

```json
{
  "name": "ahu-35pt",
  "manufacturer": "Example Corp",
  "model": "AHU-350",
  "points": [
    {
      "id": "supply-air-temp",
      "display_name": "Supply Air Temperature",
      "type": "analog-input",
      "unit": "degrees-fahrenheit",
      "protocol_binding": {
        "protocol": "bacnet",
        "config": {
          "object_type": "analog-input",
          "object_instance": 1
        }
      }
    },
    {
      "id": "fan-speed-cmd",
      "display_name": "Fan Speed Command",
      "type": "analog-output",
      "unit": "percent",
      "writable": true,
      "protocol_binding": {
        "protocol": "modbus",
        "config": {
          "register_type": "holding",
          "address": 100,
          "data_type": "uint16",
          "scale": 0.1
        }
      }
    }
  ]
}
```

### System templates

`SystemTemplate` provides pre-built site hierarchies for common building types:

- **Small Office** -- Typical 2-3 AHU office building
- **School** -- Multi-zone educational facility
- **Hospital** -- Complex multi-system healthcare facility

Templates are applied through `apply_template_iterative()` (stack-based async) and automatically tag equipment and points using `suggest_equip_tags()` and `suggest_point_tags()`.
