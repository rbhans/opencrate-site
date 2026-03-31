# Architecture Overview

OpenCrate is organized around a central platform state shared across protocol services, persistence, the API layer, and the operator UI.

## Major subsystems

- `src/platform.rs` creates the long-lived runtime services and shared handles
- `src/store/` contains SQLite-backed persistence for configuration, runtime history, and feature modules
- `src/event/` distributes internal events used by notifications, MQTT, webhooks, and other automation
- `src/api/` exposes REST and WebSocket endpoints when the `api` feature is enabled
- `src/gui/` contains the Dioxus desktop and web UI components
- `src/bridge/`, `src/protocol/`, and protocol-specific modules handle field integration

## Architectural style

The project is intentionally modular rather than microservice-based:

- one deployable Rust application
- many feature-oriented modules
- explicit persistence per domain
- event-driven reactions for cross-cutting concerns

That keeps local development and self-hosted deployment simple while still letting features evolve independently.

## Read next

- [Platform Overview](platform-overview.md)
- [API and Realtime](api-and-realtime.md)
- [Data and Persistence](data-and-persistence.md)
