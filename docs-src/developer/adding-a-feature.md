# Adding a Feature

## Recommended path

Most new feature work in OpenCrate goes more smoothly if you add it in this order:

1. domain types and persistence
2. runtime wiring in platform/shared state
3. API routes if the feature needs remote access
4. UI views if the feature is operator-facing
5. event subscribers or schedulers if it reacts asynchronously

## Why this order works

The application is organized around stable domain modules and shared runtime state. If you begin with a UI or route before the store and runtime shape are clear, you usually end up backtracking.

## Common touch points

- `src/store/` for schema and persistence
- `src/platform.rs` for runtime registration
- `src/api/routes/` for HTTP exposure
- `src/gui/components/` for operator workflows
- `src/event/` for cross-cutting reactions
