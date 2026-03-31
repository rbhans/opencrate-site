# Platform Initialization and Runtime

This document covers how OpenCrate assembles its runtime, manages shared state, monitors health, loads plugins, and shuts down.

## Platform Init

Everything starts with a single function:

```rust
pub async fn init_platform(
    scenario_path: &Path,
    profiles_dir: &Path,
) -> Result<(Platform, BridgeHandles)>
```

This function executes the full startup sequence:

1. **Load configuration.** `resolve_scenario()` reads the scenario file and all referenced device profiles from the profiles directory.
2. **Create EventBus.** Broadcast channel with 4096 capacity.
3. **Create ModelState stores.** PointStore (in-memory), NodeStore (hot cache + SQLite).
4. **Create AutomationState stores.** All 14 domain stores, each spawning a dedicated SQLite thread.
5. **Build PluginRegistry.** Register protocol plugins, history backends, alarm evaluators.
6. **Create HealthRegistry.** Empty map, components register themselves.
7. **Auto-create nodes.** `auto_create_nodes()` reads the scenario and creates equipment and point nodes with auto-tagging and protocol bindings.
8. **Start protocol bridges.** BACnet networks and Modbus bridges connect to field devices.
9. **Start background subscribers.** AlarmRouter, MqttPublisher, WebhookDispatcher, ExportPublisher, FddEngine, ExecutionEngine -- each subscribes to the EventBus.
10. **Start DiscoveryService.** Ready for user-initiated device scans.
11. **Return Platform + BridgeHandles.**

The CLI calls `init_platform` directly. The GUI has its own initialization path but creates the same components, wiring them into `AppState`.

## Platform Struct

```rust
pub struct Platform {
    pub model: ModelState,
    pub automation: AutomationState,
    pub discovery_service: DiscoveryService,
}
```

### ModelState

The foundation layer. Everything else depends on these components.

| Field | Type | Description |
|---|---|---|
| `point_store` | `PointStore` | In-memory point values and status flags |
| `node_store` | `NodeStore` | Unified object hierarchy with hot cache |
| `event_bus` | `EventBus` | Broadcast channel for all platform events |
| `plugin_registry` | `PluginRegistry` | Registered plugins (protocol, history, alarm, logic) |
| `loaded` | `LoadedScenario` | Resolved scenario config and device profiles |
| `health` | `HealthRegistry` | Per-component health status |

### AutomationState

Fourteen stores covering every automation domain.

| Field | Database | Domain |
|---|---|---|
| `alarm_store` | `data/alarms.db` | Alarm lifecycle |
| `schedule_store` | `data/schedules.db` | Time-based scheduling |
| `history_store` | `data/history.db` | Trend data |
| `entity_store` | `data/entities.db` | Legacy entities |
| `discovery_store` | `data/discovery.db` | Device/point discovery |
| `program_store` | `data/programs.db` | Logic programs |
| `notification_store` | `data/notifications.db` | Notification routing |
| `mqtt_store` | `data/mqtt.db` | MQTT config |
| `commissioning_store` | `data/commissioning.db` | Commissioning checklists |
| `report_store` | `data/reports.db` | Report definitions and runs |
| `energy_store` | `data/energy.db` | Energy meters, rates, rollups |
| `webhook_store` | `data/webhooks.db` | Webhook endpoints and deliveries |
| `fdd_store` | `data/fdd.db` | Fault detection rules and faults |
| `export_store` | `data/export.db` | External export targets |

## SharedPlatform

For the API server and GUI, the Platform is wrapped in a Clone-able handle:

```rust
pub struct SharedPlatform {
    // All stores (Clone = Arc internally)
    pub point_store: PointStore,
    pub node_store: NodeStore,
    pub event_bus: EventBus,
    // ... all 14 AutomationState stores ...

    pub bridge_registry: BridgeRegistry,
    pub health: HealthRegistry,
}
```

**BridgeRegistry** maps protocol name strings to `Box<dyn ProtocolBridgeHandle>`. When the API needs to write a point value to a field device, it looks up the protocol from the node's `ProtocolBinding`, finds the bridge in the registry, and calls `write_point()`.

All stores are internally `Arc`-wrapped, so cloning SharedPlatform is cheap and gives each Axum handler or GUI component its own reference to the same underlying data.

## Health Monitoring

The HealthRegistry provides a unified view of component health.

```rust
pub enum HealthStatus {
    Healthy,
    Degraded(String),  // Operational but with issues
    Down(String),       // Not functioning
    Unknown,            // Not yet checked
}
```

Components register and update their status. The health map is thread-safe (`Arc<RwLock<HashMap<String, HealthStatus>>>`).

Typical registered components:

- Each BACnet network (e.g., `bacnet:ip-main`)
- Each Modbus bridge (e.g., `modbus:rtu-field`)
- MQTT publisher (per broker)
- Report scheduler
- Energy rollup scheduler
- FDD engine

The API exposes health at `GET /api/system/health`, returning a JSON object with per-component status. This is designed for integration with external monitoring systems.

## Plugin System

Plugins are registered at compile time. There is no dynamic loading or unsafe FFI.

### Plugin Traits

**ProtocolPlugin** -- identifies a protocol:

- `protocol_id() -> &str` -- unique string (e.g., `"bacnet"`, `"modbus"`)
- `display_name() -> &str` -- human-readable name

**ProtocolBridgeHandle** -- controls a running bridge:

- `write_point(node_id, value, priority) -> Result<()>` -- write to field device
- `stop()` -- graceful shutdown
- `as_any() -> &dyn Any` -- downcast for protocol-specific operations

**HistoryBackend** -- pluggable trend storage.

**AlarmEvaluator** -- pluggable alarm evaluation logic. A `StandardAlarmEvaluator` is the default.

**LogicEnginePlugin** -- pluggable logic execution (the Rhai engine is the built-in implementation).

**ImportExportPlugin** -- pluggable data import/export.

### PluginRegistry

```rust
pub struct PluginRegistry {
    pub protocol_plugins: Vec<Box<dyn ProtocolPlugin>>,
    pub history_backends: Vec<Box<dyn HistoryBackend>>,
    // ... other plugin categories
}
```

Methods:

- `find_protocol(id: &str)` -- look up a protocol plugin by ID
- `protocol_ids()` -- list all registered protocol IDs

## Startup Sequence (Detailed)

```
resolve_scenario()
    |
    v
EventBus::new(4096)
    |
    v
PointStore::new() -----> NodeStore::new("data/nodes.db")
    |                         |
    v                         v
AlarmStore  ScheduleStore  HistoryStore  EntityStore  ... (14 stores)
    |
    v
PluginRegistry::new()  -->  register built-in protocols
    |
    v
HealthRegistry::new()
    |
    v
auto_create_nodes()  -->  equip + point nodes with tags + protocol bindings
    |
    v
BacnetNetworks::start()  -->  ModbusBridge::start()
    |
    v
AlarmRouter::start(&event_bus)
MqttPublisher::start(&event_bus)
WebhookDispatcher::start(&event_bus)
ExportPublisher::start(&event_bus)
FddEngine::start(&event_bus)
ExecutionEngine::start(&event_bus)
    |
    v
DiscoveryService::new()
    |
    v
Platform { model, automation, discovery_service } + BridgeHandles
```

## Shutdown Flow

Shutdown is cooperative. When the runtime receives a termination signal:

1. **Stop bridges.** Each `ProtocolBridgeHandle::stop()` is called. BACnet bridges cancel COV subscriptions and close transport connections. Modbus bridges close TCP/RTU connections.
2. **Stop subscribers.** Background tasks (AlarmRouter, MqttPublisher, etc.) drop their EventBus receivers and exit their event loops.
3. **Flush stores.** SQLite WAL checkpoints ensure all pending writes are persisted. Each store's command channel is dropped, causing its dedicated thread to exit after processing remaining commands.
4. **Drop EventBus.** With all subscribers gone, the broadcast channel is deallocated.

No data loss occurs on clean shutdown because SQLite WAL mode ensures durability of committed transactions, and each store processes its command queue to completion before its thread exits.
