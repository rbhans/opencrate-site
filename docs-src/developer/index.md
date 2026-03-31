# Developer Notes

## Codebase map

If you are orienting yourself in the repository, start with this flow:

1. `README.md` for product shape and build modes
2. `src/lib.rs` for top-level module wiring
3. `src/platform.rs` for runtime initialization
4. `src/store/` for feature persistence
5. `src/api/routes/` and `src/gui/components/` for user-facing entry points

## Good first navigation paths

- Want to understand alarms: start in `src/store/`, then `src/notification/`, `src/webhook/`, and related UI/API routes
- Want to understand protocols: start in `src/protocol/` and `src/bridge/`
- Want to understand the UI: start in `src/gui/app.rs`, `src/gui/state.rs`, and the feature views

## Writing docs going forward

Only pages under `docs/site/` are intended for public publishing. Internal notes like roadmap items and design specs should stay outside that subtree unless they are explicitly rewritten for public docs.

## Publishing flow

This private repo is the authoring source for public docs. A GitHub Action syncs `docs/site/` into `opencrate-site`, where the public Pages build assembles the landing page and `/docs/` section together.

## Extending the platform

For a practical feature development sequence, read [Adding a Feature](adding-a-feature.md).
