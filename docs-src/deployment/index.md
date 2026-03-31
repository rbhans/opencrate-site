# Deployment

OpenCrate ships as a single Rust binary with feature flags controlling which components are included. This guide covers building, configuring, running, and hardening a production deployment.

## Building

### Feature flags

| Flag | Description |
|------|-------------|
| `desktop` | Dioxus GUI (desktop application mode) |
| `api` | Axum HTTP API server |
| `web` | Web-served frontend |
| `atlas` | Atlas integration |
| `export-postgres` | PostgreSQL export connector |

Build with the features you need:

```bash
# API server only (headless)
cargo build --release --features api

# Desktop application with API
cargo build --release --features desktop,api

# Full build with PostgreSQL export
cargo build --release --features api,web,export-postgres
```

The resulting binary is at `target/release/opencrate`.

## Project directory structure

OpenCrate expects a project directory containing the scenario configuration and device profiles. All runtime data is stored in a `data/` subdirectory that is created automatically.

```
my-building/
  scenario.json          # Site configuration
  profiles/              # Device profile JSON files
  data/                  # Created at runtime
    nodes.db             # Node store
    points.db            # Point store
    history.db           # History store
    alarms.db            # Alarm store
    entities.db          # Entity store
    schedules.db         # Schedule store
    discovery.db         # Discovery store
    programs.db          # Logic programs
    notifications.db     # Notification routing
    mqtt.db              # MQTT configuration
    webhooks.db          # Webhook endpoints and delivery log
    commissioning.db     # Commissioning sessions
    reports.db           # Report definitions and schedules
    energy.db            # Energy analytics
    export.db            # Export connector state
    secret.key           # Auto-generated JWT signing key
    report_email_config.json  # Report email delivery config
```

All SQLite databases use WAL mode for concurrent read/write performance.

## Running

### Basic startup

```bash
# Run with a project directory
opencrate --project /path/to/my-building

# Or from within the project directory
cd /path/to/my-building
opencrate
```

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENCRATE_CORS_ORIGINS` | none | Comma-separated list of allowed CORS origins for the API. Example: `https://dashboard.example.com,https://admin.example.com` |
| `RUST_LOG` | `info` | Log level filter. Use `opencrate=debug` for verbose output from OpenCrate modules only. |

### Default ports

| Service | Port | Notes |
|---------|------|-------|
| HTTP API | 3000 | REST API and web frontend (when `api` or `web` features are enabled) |

### Logging

All logs go to stdout. In production, redirect to a file or use a log collector:

```bash
# Log to file
opencrate --project /path/to/my-building 2>&1 | tee -a /var/log/opencrate.log

# With systemd (logs go to journald automatically)
journalctl -u opencrate -f
```

## systemd service

Create `/etc/systemd/system/opencrate.service`:

```ini
[Unit]
Description=OpenCrate BMS
After=network.target

[Service]
Type=simple
User=opencrate
Group=opencrate
WorkingDirectory=/opt/opencrate/my-building
ExecStart=/usr/local/bin/opencrate --project /opt/opencrate/my-building
Restart=on-failure
RestartSec=5
Environment=RUST_LOG=info
Environment=OPENCRATE_CORS_ORIGINS=https://bms.example.com

# Hardening
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/opt/opencrate/my-building/data
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opencrate
```

## Reverse proxy

Place OpenCrate behind a reverse proxy for TLS termination, rate limiting, and static asset caching.

### nginx

```nginx
server {
    listen 443 ssl http2;
    server_name bms.example.com;

    ssl_certificate     /etc/letsencrypt/live/bms.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bms.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (if needed for future features)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Caddy

```
bms.example.com {
    reverse_proxy localhost:3000
}
```

Caddy handles TLS certificate provisioning automatically.

## Health checks

The API exposes a health endpoint for monitoring and load balancer checks:

```
GET /api/system/health
```

Returns HTTP 200 when the system is operational. Use this in your monitoring or container orchestration health checks:

```bash
# Simple check
curl -sf http://localhost:3000/api/system/health || echo "UNHEALTHY"

# In Docker HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -sf http://localhost:3000/api/system/health || exit 1
```

## Backup

OpenCrate includes a built-in backup system for all SQLite databases.

### Configuration

Backup interval and retention are configurable. Backups are stored in the project's `data/` directory.

### API-triggered backup

```
POST /api/system/backup
```

Triggers an immediate backup. Useful for pre-maintenance snapshots:

```bash
curl -X POST http://localhost:3000/api/system/backup \
  -H "Authorization: Bearer $TOKEN"
```

### Manual backup

Since all state is in SQLite databases with WAL mode, you can safely copy the `data/` directory while OpenCrate is running. Use SQLite's `.backup` command or `sqlite3` CLI for individual database backups:

```bash
# Stop for a clean backup
sudo systemctl stop opencrate
cp -r /opt/opencrate/my-building/data /backup/opencrate-$(date +%Y%m%d)
sudo systemctl start opencrate

# Or hot-copy individual databases
sqlite3 /opt/opencrate/my-building/data/nodes.db ".backup /backup/nodes-$(date +%Y%m%d).db"
```

## Security hardening

### JWT authentication

OpenCrate auto-generates a `data/secret.key` file on first run for signing JWT tokens. Protect this file:

```bash
chmod 600 /opt/opencrate/my-building/data/secret.key
chown opencrate:opencrate /opt/opencrate/my-building/data/secret.key
```

If you need to invalidate all existing sessions, delete `secret.key` and restart OpenCrate. A new key will be generated automatically.

### CORS

Set `OPENCRATE_CORS_ORIGINS` to restrict which domains can access the API from browsers. Without this variable, CORS headers are not sent, which means only same-origin requests work.

```bash
export OPENCRATE_CORS_ORIGINS="https://bms.example.com"
```

### Network segmentation

In a typical building automation deployment:

- The OpenCrate server sits on both the OT network (BACnet/Modbus) and the IT network (API access).
- Field protocol traffic (BACnet UDP 47808, Modbus TCP 502) should be on an isolated VLAN.
- API access (TCP 3000) should only be exposed through the reverse proxy on the IT network.
- Outbound MQTT and webhook traffic should be restricted to known broker/endpoint addresses.

### File permissions

```bash
# Restrict the project directory
chown -R opencrate:opencrate /opt/opencrate/my-building
chmod 750 /opt/opencrate/my-building
chmod 700 /opt/opencrate/my-building/data

# Serial ports for MS/TP or Modbus RTU
sudo usermod -aG dialout opencrate
```

### Firewall rules (iptables example)

```bash
# Allow API access only from reverse proxy
iptables -A INPUT -p tcp --dport 3000 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 3000 -j DROP

# Allow BACnet from OT VLAN only
iptables -A INPUT -p udp --dport 47808 -s 10.10.0.0/16 -j ACCEPT
iptables -A INPUT -p udp --dport 47808 -j DROP
```

## Updating

1. Build or download the new binary.
2. Stop the service: `sudo systemctl stop opencrate`
3. Take a backup of the `data/` directory.
4. Replace the binary.
5. Start the service: `sudo systemctl start opencrate`

SQLite database migrations run automatically on startup. There is no manual migration step.
