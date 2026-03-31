# Getting Started

This page covers everything needed to build, run, and verify a local OpenCrate instance.

## Prerequisites

**Rust toolchain** -- Install via [rustup](https://rustup.rs/). The project targets stable Rust (2021 edition). Verify with:

```
rustc --version
cargo --version
```

**System dependencies** -- On macOS, Xcode command line tools are sufficient. On Linux, you need a C compiler, pkg-config, and development headers for OpenSSL and SQLite. On Debian/Ubuntu:

```
sudo apt install build-essential pkg-config libssl-dev libsqlite3-dev
```

For the desktop UI (`desktop` feature), you also need the platform-specific Dioxus dependencies. See the [Dioxus setup guide](https://dioxuslabs.com/learn/0.6/getting_started) for your OS.

## Clone and build

```
git clone https://github.com/opencrate/opencrate-bms.git
cd opencrate-bms
cargo build --features desktop,api
```

A release build for deployment:

```
cargo build --release --features api
```

## Run modes

OpenCrate uses Cargo feature flags to control which surfaces are compiled in.

| Command | What it starts |
|---|---|
| `cargo run --features desktop,api` | Desktop window + API server on port 3000 |
| `cargo run --features desktop` | Desktop window only, no HTTP server |
| `cargo build --release --features api` | API server only (headless) |
| `cargo run --features web` | Browser-based Dioxus UI |

Additional optional features:

| Feature | Purpose |
|---|---|
| `atlas` | Enables the Haystack tag matcher for semantic point queries |
| `export-postgres` | Enables the Postgres export target for the export pipeline |

## Ports and endpoints

When the `api` feature is enabled:

| Endpoint | Description |
|---|---|
| `http://localhost:3000` | API root |
| `http://localhost:3000/health` | Health check (returns JSON with uptime and store status) |
| `ws://localhost:3000/ws` | WebSocket for live value/alarm/event updates |
| `http://localhost:3000/api/*` | REST API (80+ endpoints across all subsystems) |

The port is configured in `scenario.json` under the web server settings.

## First-run setup

On first launch, OpenCrate:

1. Reads `scenario.json` and resolves device profiles from `profiles/`.
2. Creates the `data/` directory if it does not exist.
3. Runs SQLite migrations for all stores (each store manages its own database).
4. Generates `data/secret.key` -- a random JWT signing key used for API authentication. Keep this file safe; deleting it invalidates all active sessions.
5. Prompts for (or auto-creates) an admin user stored in `data/users.db`.
6. Auto-creates nodes from the scenario (equipment and point nodes with auto-tagging and protocol bindings).
7. Starts protocol bridges (BACnet and Modbus) based on scenario configuration.
8. Starts all background services (alarm routing, MQTT publishing, webhook dispatch, report scheduling, energy rollups, FDD engine, export publisher).

## Project directory structure

A working project directory looks like this:

```
my-building/
  scenario.json          # Scenario-level settings: tick rate, protocol configs, web server port
  profiles/              # Device profile JSON files (one per device model)
    vav-controller.json
    ahu-profile.json
  data/                  # Created at runtime -- all persistent state lives here
    secret.key           # Auto-generated JWT signing key
    nodes.db             # Unified node model (site/space/equip/point hierarchy)
    alarms.db            # Active and historical alarms
    history.db           # Point value history
    schedules.db         # Weekly schedules and exception calendars
    entities.db          # Legacy entity store
    discovery.db         # Discovered devices and points
    programs.db          # Logic engine programs and execution logs
    notifications.db     # Notification recipients, routing rules, shelving, delivery log
    mqtt.db              # MQTT broker configs and topic patterns
    commissioning.db     # Commissioning sessions and checklists
    reports.db           # Report definitions, schedules, and execution history
    energy.db            # Meters, rates, baselines, and rollup data
    webhooks.db          # Webhook endpoints, delivery log, and config
    fdd.db               # Fault detection rules and faults
    export.db            # Export pipeline configuration and state
    users.db             # User accounts and roles
    audits.db            # Audit log
    overrides.db         # Point value overrides
```

Each database uses WAL mode and is managed by a dedicated store thread that accepts commands over an mpsc channel. This means stores never block each other and migrations run independently.

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `OPENCRATE_CORS_ORIGINS` | `http://localhost:3000,http://localhost:5173,http://localhost:8080` | Comma-separated list of allowed CORS origins for the API |

CORS configuration matters when running the browser UI (`web` feature) or a separate frontend against the API server.

## Boot sequence detail

When `init_platform` runs, the startup order is:

1. Resolve `scenario.json` + `profiles/` into a `LoadedScenario`.
2. Create the `EventBus` (broadcast, capacity 4096).
3. Initialize `PointStore` from device profiles.
4. Open `NodeStore` (`data/nodes.db`) and run migrations.
5. Auto-create nodes from the scenario (equipment + point nodes with auto-tagging).
6. Start 14 store threads: alarm, schedule, history, entity, discovery, program, notification, MQTT, commissioning, report, energy, webhook, FDD, export.
7. Start protocol bridges (BACnet multi-network + Modbus).
8. Seed built-in FDD rules.
9. Start background services: AlarmRouter, MqttPublisher, WebhookDispatcher, ExportPublisher, ReportScheduler, EnergyRollupScheduler, FddEngine.
10. Start DiscoveryService.
11. Wire the UI and/or API against the shared `Platform` state.

## Quick verification

After launching with `cargo run --features desktop,api`:

**Check the health endpoint:**

```
curl http://localhost:3000/health
```

You should get a JSON response with uptime and store status information.

**Check the API is responding:**

```
curl http://localhost:3000/api/nodes
```

This returns the current node tree (may require authentication depending on configuration).

**Desktop UI:** The Dioxus window should open automatically showing the operator dashboard. Use the Config tab to browse discovery, alarm routing, MQTT, webhooks, reports, energy, and commissioning settings.

## API authentication

The API uses JWT bearer tokens. Rate limiting is applied at two levels:

- **General:** 20 requests per 60 seconds per client.
- **Login:** 10 attempts per 15 minutes per IP address.

To authenticate, POST credentials to the login endpoint and use the returned token in the `Authorization: Bearer <token>` header for subsequent requests. The request body limit is 1 MB.

## Next steps

Once you have a running instance:

1. [Architecture Overview](../architecture/index.md) -- Understand the platform shape and event-driven design.
2. [Features](../features/index.md) -- Walk through the operator-facing capabilities.
3. [Integrations](../integrations/index.md) -- Configure BACnet and Modbus connections.
4. [Deployment](../deployment/index.md) -- Package for production use.
