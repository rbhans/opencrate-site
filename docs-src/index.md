# OpenCrate BMS Docs

OpenCrate is an open-source building management system built in Rust. The product combines a protocol layer, an application platform, a web/API surface, and an operator UI into one deployable system.

Use these docs to understand how the platform is put together and how the main subsystems fit.

## Start here

- Read [Getting Started](getting-started/index.md) for the fastest path to a local instance.
- Read [Architecture](architecture/index.md) for the high-level platform shape.
- Read [Features](features/index.md) for the major operator-facing capabilities.
- Read [Deployment](deployment/index.md) for packaging and runtime expectations.
- Read [Integrations](integrations/index.md) for protocols and external connectors.
- Read [Developer Notes](developer/index.md) for the codebase map and extension points.

## Platform at a glance

OpenCrate currently centers around a Rust application with these major layers:

- Protocol bridges for BACnet and Modbus, with shared abstractions under `src/protocol/` and `src/bridge/`
- Application services for alarms, schedules, history, logic, reporting, notifications, MQTT, webhook delivery, and FDD
- SQLite-backed stores under `src/store/` for persisted configuration and runtime history
- An Axum HTTP API and WebSocket stream under `src/api/`
- A Dioxus-based operator UI under `src/gui/`

## Product shape

The codebase currently supports:

- Device discovery and commissioning
- Point history, schedules, alarms, and acknowledgements
- Notification routing, webhook delivery, and MQTT publishing
- Reporting, energy tracking, and fault detection
- Desktop and API-enabled deployments from one Rust workspace

These docs focus on the current architecture reflected in this repository rather than a future roadmap.

---

*Last synced: 2026-03-31*
