# API and Real-Time Reference

OpenCrate exposes an Axum-based REST API (feature-gated with `api`) and a WebSocket endpoint for live updates. This document covers every route module, the security model, and the real-time protocol.

## API State

The API server is built from the Platform. `ApiState` is an Axum extension containing 25 fields:

- All 14 AutomationState stores
- PointStore, NodeStore, EventBus
- HealthRegistry
- UserStore, AuditStore, OverrideStore
- JWT secret
- BackupScheduler
- Rate limiters (per-IP and global)
- BridgeRegistry

## Security

### Authentication

All endpoints except `POST /api/auth/login` require a valid JWT in the `Authorization: Bearer <token>` header.

**Login flow:**

1. `POST /api/auth/login` with `{ "username": "...", "password": "..." }`
2. Server returns `{ "token": "...", "expires_at": "..." }`
3. Client includes token in all subsequent requests

Tokens carry the user's role and permissions. Expired tokens return `401 Unauthorized`.

### Authorization

Role-based access control with permissions attached to each role. Endpoints check specific permissions:

| Permission | Allowed Roles | Used By |
|---|---|---|
| `ManageNotifications` | Admin | Notification routing config |
| `ManageWebhooks` | Admin | Webhook endpoint CRUD |
| `ManageMqtt` | Admin | MQTT broker config |
| `ManageReports` | Admin, Operator | Report definitions and schedules |
| `ManageEnergy` | Admin, Operator | Energy meters and rates |
| `ManageCommissioning` | Admin, Operator | Commissioning sessions |
| `ShelveAlarm` | Admin, Operator | Alarm shelving |
| `WritePoints` | Admin, Operator | Point value writes |
| `AcknowledgeAlarms` | Admin, Operator | Alarm acknowledgment |
| `ManagePrograms` | Admin | Logic program CRUD |
| `ManageSchedules` | Admin, Operator | Schedule CRUD |
| `ManageUsers` | Admin | User CRUD |
| `ManageDiscovery` | Admin | Device discovery and acceptance |

### Rate Limiting

Two layers:

- **Per-IP rate limiter** -- prevents individual clients from overwhelming the server
- **Global rate limiter** -- protects total server capacity

Rate-limited requests receive `429 Too Many Requests` with a `Retry-After` header.

### Security Headers

Every response includes:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Strict-Transport-Security: max-age=31536000; includeSubDomains` (HSTS)
- `Referrer-Policy: strict-origin-when-cross-origin`

### CORS

Configurable CORS policy. In development, allows `localhost` origins. In production, must be explicitly configured.

### Body Limit

All requests are capped at **1 MB** to prevent abuse.

---

## Route Modules

### Points -- `/api/points`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/points` | List all points with current values |
| `GET` | `/api/points/:id` | Get a single point's value and status |
| `POST` | `/api/points/:id/write` | Write a value to a point (routes through BridgeRegistry to field device) |

### Nodes -- `/api/nodes`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/nodes` | List all nodes (filterable by type, tag, parent) |
| `GET` | `/api/nodes/:id` | Get a single node with full detail (tags, refs, properties, capabilities) |
| `POST` | `/api/nodes` | Create a node |
| `PUT` | `/api/nodes/:id` | Update a node |
| `DELETE` | `/api/nodes/:id` | Delete a node |
| `GET` | `/api/nodes/:id/children` | Get direct children of a node |
| `GET` | `/api/nodes/:id/hierarchy` | Get full subtree (recursive) |
| `GET` | `/api/nodes/by-tag/:tag` | Find nodes by tag |

### Alarms -- `/api/alarms`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/alarms` | List alarms (filterable by status, severity, device) |
| `GET` | `/api/alarms/:id` | Get alarm detail |
| `POST` | `/api/alarms/:id/acknowledge` | Acknowledge an alarm (sends BACnet ack for BACnet devices) |
| `POST` | `/api/alarms/:id/shelve` | Shelve an alarm (with expiration) |
| `POST` | `/api/alarms/:id/unshelve` | Remove shelving |
| `GET` | `/api/alarms/summary` | Alarm counts by severity and status |

### History -- `/api/history`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/history/:point_id` | Get trend data for a point (query params: start, end, interval) |
| `GET` | `/api/history/:point_id/rollup` | Get aggregated data (min, max, avg, sum over intervals) |

### Schedules -- `/api/schedules`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/schedules` | List all schedules |
| `GET` | `/api/schedules/:id` | Get schedule detail |
| `POST` | `/api/schedules` | Create a schedule |
| `PUT` | `/api/schedules/:id` | Update a schedule |
| `DELETE` | `/api/schedules/:id` | Delete a schedule |

### Discovery -- `/api/discovery`

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/discovery/scan` | Initiate a discovery scan (BACnet Who-Is, Modbus ID scan) |
| `GET` | `/api/discovery/devices` | List discovered devices |
| `GET` | `/api/discovery/devices/:id` | Get device detail with discovered points |
| `POST` | `/api/discovery/devices/:id/accept` | Accept a discovered device (creates nodes) |
| `POST` | `/api/discovery/devices/:id/ignore` | Ignore a discovered device |
| `GET` | `/api/discovery/status` | Current scan status |

### Programs -- `/api/programs`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/programs` | List all logic programs |
| `GET` | `/api/programs/:id` | Get program detail (block graph + compiled Rhai) |
| `POST` | `/api/programs` | Create a program |
| `PUT` | `/api/programs/:id` | Update a program |
| `DELETE` | `/api/programs/:id` | Delete a program |
| `POST` | `/api/programs/:id/execute` | Manually trigger program execution |
| `GET` | `/api/programs/:id/log` | Get execution log (last 100 entries) |

### Reports -- `/api/reports`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/reports` | List report definitions |
| `GET` | `/api/reports/:id` | Get report definition |
| `POST` | `/api/reports` | Create a report definition |
| `PUT` | `/api/reports/:id` | Update a report definition |
| `DELETE` | `/api/reports/:id` | Delete a report definition |
| `POST` | `/api/reports/:id/run` | Run a report now |
| `GET` | `/api/reports/:id/preview` | Preview rendered HTML |
| `GET` | `/api/reports/templates` | List available report templates |
| `GET` | `/api/reports/schedules` | List report schedules |
| `POST` | `/api/reports/schedules` | Create a report schedule |
| `PUT` | `/api/reports/schedules/:id` | Update a report schedule |
| `DELETE` | `/api/reports/schedules/:id` | Delete a report schedule |

### Energy -- `/api/energy`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/energy/meters` | List energy meters |
| `POST` | `/api/energy/meters` | Create an energy meter |
| `PUT` | `/api/energy/meters/:id` | Update an energy meter |
| `DELETE` | `/api/energy/meters/:id` | Delete an energy meter |
| `GET` | `/api/energy/rates` | List utility rate structures |
| `POST` | `/api/energy/rates` | Create a rate structure |
| `PUT` | `/api/energy/rates/:id` | Update a rate structure |
| `DELETE` | `/api/energy/rates/:id` | Delete a rate structure |
| `GET` | `/api/energy/summary` | Energy dashboard summary (consumption, demand, cost) |
| `GET` | `/api/energy/consumption` | Consumption data (filterable by meter, date range) |
| `GET` | `/api/energy/export` | Export energy data as CSV |

### Webhooks -- `/api/webhooks`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/webhooks` | List webhook endpoints |
| `POST` | `/api/webhooks` | Create a webhook endpoint |
| `PUT` | `/api/webhooks/:id` | Update a webhook endpoint |
| `DELETE` | `/api/webhooks/:id` | Delete a webhook endpoint |
| `POST` | `/api/webhooks/:id/test` | Send a test delivery |
| `GET` | `/api/webhooks/:id/deliveries` | List delivery log for an endpoint |
| `GET` | `/api/webhooks/deliveries` | List all deliveries (filterable by status) |
| `GET` | `/api/webhooks/config` | Get global webhook config (pause state) |

### FDD (Fault Detection) -- `/api/fdd`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/fdd/rules` | List fault detection rules |
| `POST` | `/api/fdd/rules` | Create a rule |
| `PUT` | `/api/fdd/rules/:id` | Update a rule |
| `DELETE` | `/api/fdd/rules/:id` | Delete a rule |
| `GET` | `/api/fdd/faults` | List active and historical faults |
| `GET` | `/api/fdd/faults/:id` | Get fault detail |

### Export -- `/api/export`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/export/targets` | List export targets (InfluxDB, Postgres) |
| `POST` | `/api/export/targets` | Create an export target |
| `PUT` | `/api/export/targets/:id` | Update an export target |
| `DELETE` | `/api/export/targets/:id` | Delete an export target |
| `POST` | `/api/export/targets/:id/test` | Test connectivity to an export target |

### Overrides -- `/api/overrides`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/overrides` | List active overrides |
| `POST` | `/api/overrides` | Create an override (with optional expiry) |
| `DELETE` | `/api/overrides/:id` | Remove an override |

### Users -- `/api/users`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/users` | List users |
| `POST` | `/api/users` | Create a user |
| `PUT` | `/api/users/:id` | Update a user |
| `DELETE` | `/api/users/:id` | Delete a user |
| `POST` | `/api/auth/login` | Authenticate and receive JWT |

### Audit -- `/api/audit`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/audit` | Query audit log (filterable by user, action, date range) |
| `GET` | `/api/audit/actions` | List all audit action types |

### System -- `/api/system`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/system/health` | Per-component health status |
| `GET` | `/api/system/info` | Version, uptime, build info |
| `POST` | `/api/system/backup` | Trigger a database backup |
| `GET` | `/api/system/backup/status` | Backup scheduler status |

---

## WebSocket -- `/ws`

The WebSocket endpoint provides real-time push updates to connected clients (the GUI and any external integrations).

### Connection

Connect to `ws://<host>:<port>/ws` with a valid JWT as a query parameter or in the initial HTTP upgrade headers.

### Message Format

All messages are JSON:

```json
{
  "type": "value_changed",
  "payload": {
    "node_id": "1001/analog-input-1",
    "value": 72.5,
    "status": "ok",
    "timestamp": "2026-03-31T14:30:00Z"
  }
}
```

### Event Types

The WebSocket forwards a subset of EventBus events to connected clients:

| Type | Description |
|---|---|
| `value_changed` | A point value was updated |
| `status_changed` | A point's status flags changed |
| `alarm_raised` | A new alarm was raised |
| `alarm_cleared` | An alarm returned to normal |
| `alarm_acknowledged` | An alarm was acknowledged |
| `device_discovered` | A new device was found during discovery |
| `device_down` | A device stopped responding |
| `device_recovered` | A previously down device came back online |
| `fdd_fault_raised` | A fault detection rule triggered |
| `fdd_fault_cleared` | A fault detection rule returned to normal |

### Subscription Filtering

Clients can send subscription messages to filter which events they receive:

```json
{
  "subscribe": {
    "node_ids": ["1001/analog-input-1", "1001/analog-input-2"],
    "event_types": ["value_changed", "alarm_raised"]
  }
}
```

Without a subscription filter, all events are forwarded.

---

## Error Responses

All error responses follow a consistent format:

```json
{
  "error": {
    "code": "not_found",
    "message": "Node with ID '999' not found"
  }
}
```

Standard HTTP status codes:

| Code | Meaning |
|---|---|
| `400` | Bad request (invalid JSON, missing fields) |
| `401` | Unauthorized (missing or expired JWT) |
| `403` | Forbidden (insufficient permissions) |
| `404` | Not found |
| `409` | Conflict (duplicate ID, concurrent modification) |
| `422` | Unprocessable entity (validation failure) |
| `429` | Rate limited |
| `500` | Internal server error |
