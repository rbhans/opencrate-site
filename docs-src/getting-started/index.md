# Getting Started

## Local run options

OpenCrate can be run in a few different shapes depending on what you want to validate:

- `cargo run --features desktop,api` for the all-in-one desktop app with embedded web server
- `cargo build --release --features api` for an API server deployment
- `cargo run --features desktop` for desktop UI only

The current repository README covers the quickest commands and expected ports.

## What boots with the app

At startup, the platform initializes core services and shared state, then conditionally starts UI and API surfaces based on enabled features.

The important runtime concerns are:

- project loading
- store initialization and migrations
- event bus startup
- background services like notification routing, reporting, export tasks, and protocol tasks
- UI/API state wiring

## Project data layout

A project directory typically contains:

- `scenario.json` for scenario-level settings
- `profiles/` for profile JSON
- `data/` for runtime SQLite databases and generated secrets

This separation lets the binary remain portable while project state lives outside the compiled application.

## Recommended first reading

After you have the app running, read:

1. [Architecture Overview](../architecture/index.md)
2. [Core Features](../features/index.md)
3. [Deployment Model](../deployment/index.md)
