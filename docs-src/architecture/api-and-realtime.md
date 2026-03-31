# API and Realtime

## HTTP surface

When the `api` feature is enabled, OpenCrate exposes an Axum-based API rooted at `/api/`.

The current API surface includes endpoints for:

- auth and first-run setup
- points, alarms, and discovery data
- reporting and exports
- energy and FDD features
- webhook and notification configuration

## Realtime updates

The WebSocket endpoint provides live updates for operator-facing screens and automation clients that need state changes without polling.

Internally, that realtime layer is fed by the same platform and event-bus updates used by other background services.

## Auth model

Authentication is handled in the application layer and guarded by role-based permissions. API routes are grouped by domain and permission requirements under `src/api/routes/`.

## Why the split matters

The API layer is deliberately thin:

- route handlers parse requests and validate access
- stores and domain modules perform the actual work
- shared state keeps UI and API behavior aligned

That makes it easier to add new routes without re-implementing business logic.
