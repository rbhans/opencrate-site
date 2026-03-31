# Data and Persistence

OpenCrate uses SQLite for all persistence. Each domain store owns its own database file, runs SQL on a dedicated thread, and communicates with the rest of the system through channels. This document covers the store pattern, every store and its schema, the event bus, and the node model.

## Store Pattern

Every store in OpenCrate follows the same architecture:

```
Caller  --mpsc::Sender<Command>-->  Dedicated Thread  --SQLite-->  data/*.db
   ^                                      |
   |                                      |
   +------oneshot::Sender<Response>-------+
```

**How it works:**

1. The store spawns a dedicated OS thread at startup.
2. That thread opens a SQLite connection (WAL mode) and runs a receive loop.
3. Callers send commands through an `mpsc::Sender<Command>`. Each command includes a `oneshot::Sender` for the response.
4. The store thread executes SQL, sends the result back through the oneshot channel.
5. The caller awaits the oneshot receiver (async-compatible).

**Why this pattern:**

- SQLite performs best with a single writer. One thread per database eliminates contention.
- WAL mode allows concurrent readers without blocking the writer.
- The mpsc channel serializes writes naturally, no mutexes needed.
- The oneshot response channel integrates cleanly with async code.

### Watch Channels

Many stores expose a `tokio::sync::watch` channel that broadcasts the latest configuration version. Subscribers (like the MqttPublisher or WebhookDispatcher) use this for **hot reload** -- when config changes, they pick up the new state without restart.

### Subscribe Channels

Some stores expose a `tokio::sync::broadcast` channel for fine-grained change notifications. The ProgramStore uses this so the ExecutionEngine knows when a program was updated and needs recompilation.

---

## All Stores

### PointStore

- **Storage**: In-memory (`HashMap` behind `Arc<RwLock>`)
- **No database file** -- points are transient runtime values
- **EventBus integration**: Publishes `ValueChanged` and `StatusChanged` events
- **COV filtering**: `set_if_changed()` skips events when the value is unchanged (used by Modbus poll loop)

### NodeStore

- **Database**: `data/nodes.db`
- **Tables**: `node`, `node_tag`, `node_ref`, `node_property`, `node_capability`, `protocol_binding`
- **Hot cache**: `Arc<RwLock<HashMap<NodeId, NodeSnapshot>>>` for fast reads
- **Hierarchy queries**: Recursive CTEs for subtree traversal
- **EventBus**: Publishes `EntityCreated`, `EntityUpdated`, `EntityDeleted`

### HistoryStore

- **Database**: `data/history.db`
- **Tables**: Trend data indexed by point ID and timestamp
- **Rollup support**: Pre-aggregated intervals (min, max, avg, sum)

### AlarmStore

- **Database**: `data/alarms.db`
- **Tables**: Active alarms, alarm history
- **EventBus**: Publishes `AlarmRaised`, `AlarmCleared`, `AlarmAcknowledged`
- **Flag sync**: EventBus-driven (replaced old 3-second poll loop); stale check remains periodic (30s)

### ScheduleStore

- **Database**: `data/schedules.db`
- **Tables**: Schedule definitions, schedule entries
- **EventBus**: Publishes `ScheduleWritten`

### EntityStore

- **Database**: `data/entities.db`
- **Tables**: Legacy entity records
- **EventBus**: Publishes `EntityCreated`, `EntityUpdated`, `EntityDeleted`

### DiscoveryStore

- **Database**: `data/discovery.db`
- **Tables**: `discovered_device`, `discovered_point`
- **Migrations**: v3 added `network_id` column, v4 added `object_list_stale` column
- **Device states**: Discovered, Accepted, Ignored

### ProgramStore

- **Database**: `data/programs.db`
- **Tables**: Program definitions, execution log (auto-pruned to last 100 per program)
- **Subscribe channel**: Notifies ExecutionEngine of program changes

### NotificationStore

- **Database**: `data/notifications.db`
- **Tables**: `recipient`, `routing_rule`, `alarm_shelving`, `notification_log`
- **Watch channel**: AlarmRouter hot-reloads routing rules on change

### MqttStore

- **Database**: `data/mqtt.db`
- **Tables**: `mqtt_broker`, `mqtt_topic_pattern`
- **Watch channel**: MqttPublisher hot-reloads broker configs on change

### CommissioningStore

- **Database**: `data/commissioning.db`
- **Tables**: `commission_session`, `commission_item`
- **Session lifecycle**: not_started -> in_progress -> completed -> signed_off
- **Item lifecycle**: not_started -> verified / failed / deferred

### ReportStore

- **Database**: `data/reports.db`
- **Tables**: `report_definition`, `report_schedule`, `report_execution`
- **Cascade deletes**: Deleting a definition removes its schedules and execution history

### EnergyStore

- **Database**: `data/energy.db`
- **Tables**: `utility_rate`, `energy_meter`, `energy_baseline`, `energy_rollup`
- **Upsert on conflict**: Partial rollups are updated in place when final data arrives

### WebhookStore

- **Database**: `data/webhooks.db`
- **Tables**: `webhook_endpoint`, `webhook_delivery`, `webhook_config`
- **Watch channel**: WebhookDispatcher hot-reloads endpoint configs on change

### FddStore

- **Database**: `data/fdd.db`
- **Tables**: Fault detection rules, fault instances
- **EventBus**: FddEngine publishes `FddFaultRaised`, `FddFaultCleared`

### ExportStore

- **Database**: `data/export.db`
- **Tables**: Export target configurations, export state tracking
- **Watch channel**: ExportPublisher hot-reloads target configs on change

### UserStore

- **Database**: `data/users.db`
- **Tables**: Users, roles, password hashes
- **Password storage**: Argon2 hashing

### AuditStore

- **Database**: `data/audit.db`
- **Tables**: Audit log entries (user, action, target, timestamp, detail)
- **Write-only from application perspective**: No updates or deletes, append-only log

### OverrideStore

- **Database**: `data/overrides.db`
- **Tables**: Active overrides with optional expiry timestamps

---

## Event Bus

The EventBus is a `tokio::sync::broadcast` channel with capacity **4096**. Events are wrapped in `Arc<Event>` for zero-copy distribution to all subscribers.

### All 18 Event Variants

| Event | Published By | Payload |
|---|---|---|
| **ValueChanged** | PointStore | node_id, value, timestamp |
| **StatusChanged** | PointStore | node_id, old_status, new_status |
| **AlarmRaised** | AlarmStore | alarm_id, node_id, severity, message |
| **AlarmCleared** | AlarmStore | alarm_id, node_id |
| **AlarmAcknowledged** | AlarmStore | alarm_id, user |
| **ScheduleWritten** | ScheduleStore | schedule_id, node_id, value |
| **EntityCreated** | EntityStore / NodeStore | entity_id, entity_type |
| **EntityUpdated** | EntityStore / NodeStore | entity_id, changed_fields |
| **EntityDeleted** | EntityStore / NodeStore | entity_id |
| **DeviceDiscovered** | DiscoveryService / Bridges | device_key, protocol, network_id |
| **DeviceDown** | Bridges (after backoff exhaustion) | device_key, protocol, error |
| **DeviceRecovered** | Bridges (on reconnect) | device_key, protocol |
| **DeviceAccepted** | DiscoveryService | device_key, created_node_ids |
| **DiscoveryScanComplete** | DiscoveryService | protocol, device_count, point_count |
| **DeviceMonitorCycle** | BACnet monitor loop | checked_count, online_count, offline_count |
| **ObjectListChanged** | BACnet monitor loop | device_key, old_count, new_count |
| **FddFaultRaised** | FddEngine | fault_id, rule_id, node_id, severity, message |
| **FddFaultCleared** | FddEngine | fault_id, rule_id, node_id |

### Subscriber Wiring

Each subscriber calls `event_bus.subscribe()` to get a `broadcast::Receiver<Arc<Event>>`, then runs an async loop that matches on event variants it cares about:

```
EventBus (broadcast, capacity 4096)
    |
    +---> AlarmRouter       [AlarmRaised, AlarmCleared, AlarmAcknowledged]
    +---> MqttPublisher     [ValueChanged, AlarmRaised, AlarmCleared, DeviceDown, DeviceDiscovered]
    +---> WebhookDispatcher [AlarmRaised, AlarmCleared, AlarmAcknowledged, DeviceDown, DeviceRecovered, FddFaultRaised, FddFaultCleared]
    +---> ExportPublisher   [ValueChanged, AlarmRaised, AlarmCleared]
    +---> FddEngine         [ValueChanged]
    +---> ExecutionEngine   [ValueChanged (for OnChange-triggered programs)]
```

### Backpressure

If a subscriber falls 4096 messages behind, the broadcast channel returns a `Lagged` error. Subscribers handle this by logging the lag count and continuing from the latest message. No events are queued indefinitely -- this is a deliberate design choice to prevent memory growth from slow consumers.

---

## Node Model

The node model is a unified hierarchy that represents every physical and logical object in a building.

### Node Structure

```rust
pub struct Node {
    pub id: NodeId,              // e.g., "1001/analog-input-1" or "1001"
    pub node_type: NodeType,     // Site, Space, Equip, Point, VirtualPoint
    pub dis: String,             // Display name
    pub parent_id: Option<NodeId>,
    pub value: Option<PointValue>,
    pub status: PointStatus,
    pub tags: HashSet<String>,
    pub refs: HashMap<String, NodeId>,
    pub properties: HashMap<String, serde_json::Value>,
    pub capabilities: NodeCapabilities,
    pub protocol_binding: Option<ProtocolBinding>,
}
```

### Node Types

| Type | Description | Example |
|---|---|---|
| **Site** | A building or campus | "Main Office" |
| **Space** | A zone, floor, or room | "Floor 2", "Room 201" |
| **Equip** | A piece of equipment | "AHU-1", "Boiler-3" |
| **Point** | A physical I/O point bound to a protocol | "Supply Air Temp" |
| **VirtualPoint** | A calculated or software-only point | "Avg Zone Temp" |

### NodeCapabilities

Boolean flags indicating what a node supports:

```rust
pub struct NodeCapabilities {
    pub readable: bool,
    pub writable: bool,
    pub historizable: bool,
    pub alarmable: bool,
    pub schedulable: bool,
}
```

### ProtocolBinding

How a node maps to a field device:

```rust
pub struct ProtocolBinding {
    pub protocol: String,           // "bacnet", "modbus", etc.
    pub config: serde_json::Value,  // Protocol-specific config
}
```

The `config` field is intentionally untyped JSON. This keeps the node model protocol-agnostic while allowing protocol-specific details:

**BACnet binding config:**
```json
{
  "device_instance": 1001,
  "object_type": "analog-input",
  "object_instance": 1,
  "network_id": "ip-main"
}
```

**Modbus binding config:**
```json
{
  "device_id": "meter-1",
  "register_type": "holding",
  "address": 100,
  "data_type": "float32",
  "byte_order": "big-endian"
}
```

Convenience constructors `ProtocolBinding::bacnet()`, `::modbus()`, and `::virtual_binding()` build these for common cases. A custom `Deserialize` implementation handles the legacy tagged-enum format for backward compatibility.

### Node ID Convention

- Points: `"{device_instance_id}/{point_id}"` (e.g., `"1001/analog-input-1"`)
- Equipment: `"{device_instance_id}"` (e.g., `"1001"`)
- Sites and Spaces: user-defined string IDs

### Hierarchy

Nodes form a tree through `parent_id`. The NodeStore supports recursive CTE queries to fetch entire subtrees efficiently. A typical hierarchy:

```
Site: "Main Office"
  Space: "Floor 1"
    Equip: "AHU-1" (device 1001)
      Point: "1001/analog-input-1" (Supply Air Temp)
      Point: "1001/analog-input-2" (Return Air Temp)
      Point: "1001/analog-output-1" (Damper Position)
    Equip: "VAV-101" (device 1002)
      Point: "1002/analog-input-1" (Zone Temp)
      Point: "1002/analog-output-1" (Airflow Setpoint)
  Space: "Floor 2"
    ...
```

### Auto-Tagging

When nodes are created from scenario configs or accepted from discovery, `suggest_equip_tags()` and `suggest_point_tags()` apply heuristic tags based on names and object types (e.g., an object named "Supply Air Temp" gets tags `air`, `temp`, `sensor`, `supply`). These tags drive alarm routing rules, report filters, and energy meter associations.
