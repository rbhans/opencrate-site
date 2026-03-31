# Architecture Overview

OpenCrate BMS is structured as a layered, event-driven platform. Every subsystem communicates through a central event bus, stores manage their own SQLite databases on dedicated threads, and the entire runtime is assembled by a single `init_platform` call.

## Three-Layer Model

The platform is organized into three layers, each building on the one below it.

### Layer 1: ModelState

The foundation. Owns the core data structures that every other layer reads and writes.

| Component | Purpose |
|---|---|
| **PointStore** | In-memory thread-safe point values and status |
| **NodeStore** | Unified object model (sites, spaces, equipment, points) with hot cache + SQLite |
| **EventBus** | Broadcast channel (4096 capacity) distributing `Arc<Event>` to all subscribers |
| **PluginRegistry** | Protocol plugins, history backends, alarm evaluators registered at startup |
| **LoadedScenario** | The resolved scenario config and device profiles |
| **HealthRegistry** | Thread-safe health status map (Healthy, Degraded, Down, Unknown) |

### Layer 2: AutomationState

Fourteen domain-specific stores, each running its own SQLite database on a dedicated thread. These stores implement the building automation logic: alarms, schedules, history, energy analytics, fault detection, reporting, and integrations.

| Store | Domain |
|---|---|
| alarm_store | Alarm lifecycle and acknowledgment |
| schedule_store | Time-based scheduling |
| history_store | Point value history (trend data) |
| entity_store | Legacy entity persistence |
| discovery_store | Protocol device/point discovery state |
| program_store | Logic engine programs and execution logs |
| notification_store | Alarm routing recipients, rules, shelving |
| mqtt_store | MQTT broker configs and topic patterns |
| commissioning_store | Device commissioning sessions and checklists |
| report_store | Report definitions, schedules, executions |
| energy_store | Meters, rates, baselines, rollups |
| webhook_store | Webhook endpoints and delivery logs |
| fdd_store | Fault detection rules and fault history |
| export_store | External export targets (InfluxDB, Postgres) |

### Layer 3: Platform

The assembled runtime. `init_platform` wires everything together and returns:

- **Platform** containing ModelState, AutomationState, and the DiscoveryService
- **BridgeHandles** containing active protocol bridges (BACnet networks, Modbus bridges)

For concurrent access from the API server and GUI, Platform is wrapped in **SharedPlatform** -- a Clone-able struct that includes a **BridgeRegistry** mapping protocol strings to `Box<dyn ProtocolBridgeHandle>`.

## Event-Driven Design

The EventBus is the nervous system. When a point value changes, an alarm fires, or a device goes offline, an event is broadcast to all subscribers. No subsystem polls another -- they react.

Six permanent subscribers run as background tasks:

| Subscriber | Reacts To | Action |
|---|---|---|
| **AlarmRouter** | AlarmRaised, AlarmCleared, AlarmAcknowledged | Dispatches notifications (email, SMS, webhook) |
| **MqttPublisher** | ValueChanged, AlarmRaised, AlarmCleared, DeviceDown, DeviceDiscovered | Publishes JSON to MQTT brokers |
| **WebhookDispatcher** | AlarmRaised, AlarmCleared, AlarmAcknowledged, DeviceDown, DeviceRecovered, FddFaultRaised, FddFaultCleared | Sends HTTP POSTs to registered endpoints |
| **ExportPublisher** | ValueChanged, AlarmRaised, AlarmCleared | Writes to InfluxDB and/or Postgres |
| **FddEngine** | ValueChanged | Evaluates fault detection rules |
| **ExecutionEngine** | ValueChanged (OnChange triggers) | Runs logic programs |

## Module Map

```
src/
  platform.rs          # init_platform, Platform, SharedPlatform
  lib.rs               # Feature gates, public API surface

  event/
    bus.rs             # EventBus, Event enum (18 variants)

  node/
    mod.rs             # Node, NodeType, NodeCapabilities, ProtocolBinding

  store/               # 20 stores (mpsc + SQLite pattern)
    point_store.rs     # In-memory point values
    node_store.rs      # Node hierarchy (hot cache + SQLite)
    history_store.rs   # Trend data
    alarm_store.rs     # Alarms
    schedule_store.rs  # Schedules
    entity_store.rs    # Legacy entities
    discovery_store.rs # Discovery state
    program_store.rs   # Logic programs
    notification_store.rs
    mqtt_store.rs
    commissioning_store.rs
    report_store.rs
    energy_store.rs
    webhook_store.rs
    fdd_store.rs
    export_store.rs
    user_store.rs
    audit_store.rs
    override_store.rs

  plugin/
    mod.rs             # PluginRegistry, traits

  bridge/
    traits.rs          # PointSource trait
    bacnet.rs          # BACnet bridge (multi-network)
    modbus.rs          # Modbus bridge (TCP + RTU)
    backoff.rs         # Shared exponential backoff

  protocol/
    mod.rs             # RawProtocolValue, ProtocolDriver, ValueSink, ProfileNormalizer

  discovery/
    mod.rs             # DiscoveryService, DiscoveredDevice, DiscoveredPoint
    bacnet_adapter.rs  # BACnet -> generic discovery
    bacnet_units.rs    # ASHRAE unit mapping

  api/
    mod.rs             # Axum router, ApiState, middleware
    routes/            # 17 route modules

  gui/
    app.rs             # Dioxus desktop app
    state.rs           # AppState
    components/        # UI components

  logic/
    mod.rs             # Block types, DAG model
    compiler.rs        # Topological sort -> Rhai codegen
    engine.rs          # ExecutionEngine (periodic + event-driven)
    store.rs           # ProgramStore

  notification/
    router.rs          # AlarmRouter
    channels/          # Email, SMS, webhook channels

  mqtt/
    publisher.rs       # MqttPublisher
    topic.rs           # Topic template engine

  webhook/
    dispatcher.rs      # WebhookDispatcher
    model.rs           # Endpoint, delivery models
    providers.rs       # Slack, Teams, PagerDuty, ntfy formatters

  reporting/
    engine.rs          # ReportEngine (7 section types)
    renderer.rs        # HTML renderer (inline CSS, dark theme)
    templates.rs       # 4 built-in templates

  energy/
    consumption.rs     # Trapezoidal kW -> kWh
    demand.rs          # 15-min sliding window peak
    degree_days.rs     # HDD/CDD regression
    cost.rs            # Rate structures
    rollup.rs          # Daily/monthly orchestration
    scheduler.rs       # 15-min background cycle

  fdd/                 # Fault Detection & Diagnostics
  export/              # External data export (InfluxDB, Postgres)

  config/
    profile.rs         # DeviceProfile, Point definitions
    scenario.rs        # ScenarioConfig, DeviceInstance
    loader.rs          # resolve_scenario()
    template.rs        # SystemTemplate, auto_create_nodes, auto-tagging

  auth.rs              # Permissions, JWT, roles
```

## Design Principles

**No dynamic loading.** All plugins and protocol drivers are registered at compile time via the PluginRegistry. This keeps the binary self-contained and avoids unsafe FFI.

**One store, one database.** Each store owns a single SQLite file. No store reads another store's database. Cross-store coordination happens through the EventBus.

**Zero-copy events.** Events are wrapped in `Arc<Event>` so broadcast to N subscribers does not clone payloads.

**Graceful degradation.** HealthRegistry tracks per-component status. If a BACnet network goes down, the rest of the platform continues operating. Bridge reconnection uses exponential backoff.

**Feature gates.** The API server (`api` feature) and GUI (`gui` feature) are optional. The core platform can run headless.

## Read next

- [Platform Overview](platform-overview.md)
- [API and Realtime](api-and-realtime.md)
- [Data and Persistence](data-and-persistence.md)
