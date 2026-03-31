# Data and Persistence

## Store model

Most major features in OpenCrate have their own store under `src/store/`. Each store owns its schema, migrations, and CRUD/query helpers for one domain.

Examples include:

- alarms
- reporting
- notifications
- commissioning
- MQTT
- energy
- FDD
- export

## Why SQLite per domain works here

OpenCrate is optimized for self-hosted deployments and local-first development. SQLite keeps packaging simple while still supporting a fairly rich application model.

Benefits of the current approach:

- no external database required
- transactional writes for config and runtime history
- predictable deployment footprint
- feature modules can evolve their schema independently

## Event-driven reactions

Persistence is only part of the story. Many writes emit or feed events that other subsystems consume, which is how alarms can trigger notifications, webhooks, and MQTT publishes without hard-coding those concerns into every store.

## Developer takeaway

When you need to understand a feature, start with its store and then follow:

1. the route or UI component that calls it
2. any event-bus subscribers that react to it
3. any background worker that schedules or retries work
