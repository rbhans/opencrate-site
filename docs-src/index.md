# OpenCrate BMS

OpenCrate is an open-source building automation system written in Rust. It ships as a single binary that combines protocol bridges, application services, a REST/WebSocket API, and an operator desktop UI. The goal is to provide a credible open-source alternative to proprietary BAS platforms like Niagara.

## Capabilities

**Protocol integration** -- BACnet (IP, MS/TP, SC, BBMD/Foreign) and Modbus (TCP, RTU) with multi-network support, device discovery, COV subscriptions, priority-array writes, trend-log backfill, FC8/FC22/FC23/FC24/FC43 Modbus diagnostics, and a profile library for auto-mapping registers.

**Point management** -- Unified node model (Site/Space/Equip/Point/VirtualPoint) with tag-based querying, protocol bindings, history, overrides, and COV-filtered change detection.

**Alarms and notifications** -- Alarm evaluation with severity and shelving, rule-based notification routing with tiered escalation, exponential retry, and delivery via webhook, email (SMTP or HTTP), and SMS (Twilio or generic POST).

**Scheduling** -- Weekly schedules and exception calendars with BACnet device sync (read/write from remote schedule objects).

**Logic engine** -- Visual block programming compiled to Rhai scripts. Block types include math, logic, compare, timing, PID, point read/write, and custom script. Programs execute on a 1-second tick with event-driven triggers.

**Reporting** -- Seven report section types, four built-in templates, inline-CSS HTML rendering, scheduled SMTP delivery, and on-demand generation via API or UI.

**Energy analytics** -- Trapezoidal consumption calculation, 15-minute sliding-window demand, degree-day regression with CV-RMSE/NMBE, flat/TOU/tiered/demand cost rates, and daily/monthly rollups on a 15-minute background cycle.

**Fault detection and diagnostics** -- Rule-based FDD engine with built-in seed rules, fault store, and operator UI.

**MQTT publishing** -- Per-broker event loops with auto-reconnect, template-based topic engine, and hot-reload configuration.

**Webhook delivery** -- Provider-specific formatters for Slack (Block Kit), Teams (Adaptive Cards), PagerDuty (Events API v2), ntfy, and generic JSON with HMAC-SHA256 signing. Per-endpoint event toggles, severity/tag filters, retry with backoff, and global pause.

**Export** -- Pluggable export pipeline (Postgres target available via `export-postgres` feature).

**Commissioning** -- Per-device session workflow with auto-generated checklists (read-verify, write-verify, alarm-verify, schedule-verify) and status lifecycle tracking.

**Auth and audit** -- JWT authentication with per-IP rate limiting, role-based permissions, and a full audit log.

## Architecture at a glance

| Dimension | Detail |
|---|---|
| Language | Rust (2021 edition) |
| Binary | Single binary, feature-gated surfaces |
| UI toolkit | Dioxus (desktop and browser targets) |
| API framework | Axum with tower middleware |
| Persistence | SQLite per domain (17 databases under `data/`) |
| Event system | Broadcast bus, capacity 4096, `Arc<Event>` |
| Store pattern | mpsc command channel + dedicated SQLite thread (20 stores) |
| API surface | 80+ REST endpoints, WebSocket at `/ws` |
| GUI components | 46 components under `src/gui/` |
| Modules | 25 (see module map below) |

## Module map

```
Protocol layer        bridge/  protocol/  discovery/
Node model            node/  store/  config/
Application services  energy/  fdd/  logic/  reporting/  notification/  mqtt/  webhook/  export/
Scheduling & alarms   (within store/ and event/)
Infrastructure        event/  plugin/  platform/  auth/  health/  backup/  project/  weather/
Haystack              haystack/  atlas/ (optional)
Surfaces              gui/ (optional)  api/ (optional)
```

## Documentation sections

- [Getting Started](getting-started/index.md) -- Build, run, and verify a local instance.
- [Architecture](architecture/index.md) -- Platform shape, event bus, store pattern, node model.
- [Features](features/index.md) -- Operator-facing capabilities in detail.
- [Integrations](integrations/index.md) -- BACnet, Modbus, MQTT, webhooks, and export targets.
- [Deployment](deployment/index.md) -- Packaging, configuration, and runtime expectations.
- [Developer Notes](developer/index.md) -- Codebase map, extension points, and contribution guide.

---

*Last synced: 2026-03-31*
