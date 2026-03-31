# Deployment Model

## Main deployment shapes

OpenCrate currently supports:

- desktop app with embedded web server
- API-only server deployment
- web-target client builds alongside an API backend

## Packaging assumptions

The project is designed to run with a project directory containing configuration, profiles, and runtime data. That makes upgrade and backup workflows simpler because the binary and the project data are separate concerns.

## Reverse proxy and TLS

For production-style deployments, place OpenCrate behind a reverse proxy for TLS termination. The repository already includes proxy examples under `deploy/`.

## Operational concerns

A production-minded deployment should think about:

- backups of the project data directory
- log capture and retention
- port exposure and CORS configuration
- secret handling for API auth, webhook providers, SMTP, and similar integrations

## Current tradeoffs

The system favors a simple single-node deployment over distributed infrastructure. That keeps setup approachable, but it also means scaling strategies are currently based on one application instance per project.
