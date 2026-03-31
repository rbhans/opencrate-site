# Features

OpenCrate is a full-featured Building Management System. This page maps every capability to the workflow it supports. Detailed coverage lives in the linked pages.

## Device onboarding

Getting field devices into the system and verifying they work correctly.

- **Discovery** -- Protocol-agnostic scanning for BACnet and Modbus devices. Devices move through Discovered, Accepted, and Ignored states. Scans are always user-initiated and re-discovery preserves prior state. Auto-naming heuristics and equipment grouping suggestions speed up large sites. [Details](operations.md#discovery)
- **Commissioning** -- Per-device sessions with per-point checklists (ReadVerify, WriteVerify, AlarmVerify, ScheduleVerify). Sessions progress from not_started through in_progress and completed to signed_off. [Details](operations.md#commissioning)

## Day-to-day operations

Monitoring, responding to events, and controlling equipment.

- **Alarms** -- Full lifecycle management: raised, cleared, acknowledged. Shelving with automatic expiry. EventBus-driven flag sync with periodic stale checks. Active and history views with export. [Details](operations.md#alarms)
- **Schedules** -- Weekly and exception schedules with point assignments. BACnet schedule synchronization reads and writes schedules, calendars, and exceptions directly from field devices. [Details](operations.md#schedules)
- **History** -- Time-series sample collection in interval and COV modes. Query by time range and export to CSV. [Details](operations.md#history)

## Automation

Custom control logic and fault detection running continuously in the background.

- **Logic Engine** -- Visual block programming with 19+ block types compiled to sandboxed Rhai scripts. Runs on a 1-second tick, on value change, or on a custom period. [Details](automation-and-analysis.md#logic-engine)
- **FDD (Fault Detection & Diagnostics)** -- Rule-based fault detection with threshold, stuck value, short-cycling, schedule deviation, and demand-sensitive conditions. EventBus subscriber with 30-second periodic evaluation. [Details](automation-and-analysis.md#fault-detection--diagnostics)

## Analysis and reporting

Turning operational data into insight.

- **Reporting** -- 7 section types, 4 built-in templates, scheduled SMTP delivery. Queries across history, alarms, points, and nodes. Inline-CSS HTML output that works in any email client. [Details](automation-and-analysis.md#reporting)
- **Energy Analytics** -- Consumption calculation, demand profiling, degree-day analysis, cost modeling, and automated daily/monthly rollups. Background scheduler runs every 15 minutes. [Details](automation-and-analysis.md#energy-analytics)

## Integration

Connecting OpenCrate to external systems.

- **Webhooks** -- Outbound HTTP notifications for alarm and device events. Provider-specific formatters for Slack, Teams, PagerDuty, and ntfy. HMAC-SHA256 signing, retry logic, and per-endpoint event toggles. [Details](operations.md#alarm-routing--notifications)
- **MQTT** -- Publishes point values, alarms, and device status to one or more MQTT brokers. Configurable topic templates with hot-reload. [Details](operations.md#alarm-routing--notifications)
- **Notification routing** -- Alarm-driven dispatch to webhooks, email (SMTP), and SMS (Twilio). Rule-based filtering by severity, device, and alarm type with tiered escalation and retry. [Details](operations.md#alarm-routing--notifications)
- **Export** -- Continuous or backfill export to InfluxDB and PostgreSQL. Per-connector event toggles and status tracking. [Details](automation-and-analysis.md#export)

## Administration

Security, audit, and system maintenance.

- **Auth** -- Argon2 password hashing, JWT tokens. Three roles (Admin, Operator, Viewer) with 16 granular permissions. First-run setup flow. [Details](operations.md#auth--audit)
- **Audit** -- Every operator action logged with timestamp, user, action, and details. [Details](operations.md#auth--audit)
- **Weather** -- Pluggable weather providers (OpenWeatherMap, custom HTTP). Forecast models with auto-geocoding. [Details](operations.md#weather)
- **Backup** -- Scheduled backups with configurable interval and retention. API-triggered or automatic. [Details](operations.md#backup)
