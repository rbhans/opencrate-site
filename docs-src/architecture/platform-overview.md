# Platform Overview

## Runtime composition

The platform layer is responsible for constructing the shared runtime used everywhere else in the application.

The rough startup flow is:

1. Load configuration and project context
2. Open stores and run schema initialization
3. Build shared state objects and event channels
4. Start background workers and schedulers
5. Expose UI and API layers against the same platform state

## Why this matters

This design means features such as notifications, reporting, MQTT publishing, and FDD can react to the same underlying events and stores instead of duplicating transport or persistence logic.

## Key code areas

- `src/platform.rs`
- `src/lib.rs`
- `src/event/`
- `src/store/`

## Cross-cutting services

Several features are wired as subscribers or workers around the same core runtime:

- notification routing
- webhook dispatch
- MQTT publishing
- reporting and export jobs
- discovery and commissioning state
- weather and energy services

That shared-runtime approach is one of the main reasons the codebase stays navigable even as feature count grows.
