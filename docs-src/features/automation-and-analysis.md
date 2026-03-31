# Automation and Analysis

## Logic engine

OpenCrate includes a visual logic system that compiles user-authored block diagrams into sandboxed scripts. The goal is to keep editing approachable while running a real automation layer underneath.

### Block types

Programs are directed acyclic graphs of typed blocks connected by wires. Each block has typed inputs and outputs. The full set of available blocks:

**Input/output blocks** read and write the live point database:

| Block | Purpose |
|---|---|
| PointRead | Read a live value from any node |
| PointWrite | Write a value to any node |
| VirtualPoint | Read or write a virtual (software-only) point |

**Math blocks** perform arithmetic on numeric inputs:

| Block | Operations |
|---|---|
| Math | add, sub, mul, div, min, max, abs, clamp |
| Scale | Linear scaling (slope + offset) |
| RampLimit | Limit rate of change per execution cycle |

**Logic blocks** operate on boolean values:

| Block | Operations |
|---|---|
| Logic | and, or, not, xor |
| Compare | gt, lt, gte, lte, eq, neq |
| Select | Choose one of N inputs based on a selector |
| Latch | Set/reset latch with boolean inputs |
| OneShot | Single-pulse output on rising edge |

**Timing blocks** introduce time-dependent behavior:

| Block | Purpose |
|---|---|
| Timing | delay_on, delay_off, moving_avg, rate_of_change |
| PID | Proportional-integral-derivative controller |

**Action blocks** trigger side effects:

| Block | Purpose |
|---|---|
| AlarmTrigger | Raise or clear an alarm based on input |
| Log | Write a message to the execution log |
| CustomScript | Inline Rhai code for anything the built-in blocks do not cover |

The `Constant` block provides literal values (numeric, boolean, string) as inputs to other blocks.

### Compiler

The compiler performs a topological sort using Kahn's algorithm to determine execution order, then generates Rhai v1 code from the sorted block list. Each block type has a code generation template. For advanced users, the `rhai_override` field on a program bypasses the compiler entirely and runs a hand-written Rhai script.

### Sandboxing

Generated scripts run in the embedded Rhai engine (v1, compiled with the `sync` feature for `Send + Sync` compatibility). An operation limit of 10,000 prevents runaway scripts from blocking the system.

### Triggers and execution

The `ExecutionEngine` runs as a tokio task. It uses `select!` to wake on three sources:

1. **Periodic tick** -- Default 1-second interval, configurable per program in milliseconds.
2. **OnChange** -- The engine subscribes to the EventBus and re-runs the program immediately when any node in its watch list changes value.
3. **Store version changes** -- When a program is edited, the engine picks up the new version on the next cycle via the `ProgramStore` subscription.

Each program maintains persistent state across executions through `state_get` and `state_set` functions exposed to Rhai. This enables blocks like delay timers and moving averages that need to remember values between runs.

### Point interaction

All point access is protocol-agnostic. `read(node_id)` and `write(node_id, value)` operate on the `PointStore`, which handles the translation to and from protocol-specific bridges.

### Storage

The `ProgramStore` uses the standard mpsc + SQLite thread pattern, persisting to `data/programs.db`. It stores program definitions (blocks, wires, metadata) and an execution log. The log is auto-pruned to the last 100 entries per program.

---

## Reporting

### Report engine

The `ReportEngine` pulls data from four stores -- `HistoryStore`, `AlarmStore`, `PointStore`, and `NodeStore` -- and assembles it into sections. A single report definition can contain any combination of section types.

### Section types

| Section | What it shows |
|---|---|
| HistorySummary | Aggregated statistics (min, max, average, sum) for selected points over the time range |
| AlarmSummary | Alarm counts grouped by severity, type, or device |
| AlarmList | Individual alarm records with timestamps and details |
| CurrentValues | Live snapshot of point values at report generation time |
| RuntimeSummary | Equipment on/off runtime hours and percentages |
| EnergyConsumption | Energy usage totals from the energy analytics system |
| DemandSummary | Peak demand figures with load factor |

### Point selection

Reports select points through four methods:

- **Explicit list** -- Specific node IDs.
- **By tag** -- All points matching a tag query.
- **By parent node** -- All points under a given site, space, or equipment node.
- **By device list** -- All points belonging to specified devices.

### Time ranges

Each report specifies a time range in one of three forms:

- **Absolute** -- Fixed start and end timestamps.
- **Relative** -- Last N hours or last N days from execution time.
- **Last month** -- The previous calendar month.

### Rendering

The renderer produces self-contained HTML with inline CSS. The output uses a dark theme and contains no external dependencies -- no linked stylesheets, no JavaScript, no images. This makes it safe to embed directly in email bodies.

### Built-in templates

Four templates ship by default:

| Template | Sections included |
|---|---|
| EnergySummary | EnergyConsumption, DemandSummary, HistorySummary |
| AlarmSummary | AlarmSummary, AlarmList |
| ComfortCompliance | HistorySummary (temperature/humidity points), CurrentValues |
| EquipmentRuntime | RuntimeSummary, HistorySummary |

The **Custom** template type lets operators define their own section combinations.

### Scheduling and delivery

The report scheduler ticks every 60 seconds and checks for reports that are due. When a report is due, the engine generates it and delivers it via SMTP using the lettre library. SMTP settings are read from `data/report_email_config.json`.

### Storage and API

The `ReportStore` persists to `data/reports.db` with three tables: `report_definition`, `report_schedule`, and `report_execution`. Deleting a definition cascades to its schedules and executions.

The GUI provides a Reports section under Config with three sub-tabs: Templates, Schedules, and History. A Run Now button generates the report immediately. A preview modal shows the rendered HTML before sending.

Twelve REST endpoints at `/api/reports/*` provide full CRUD for definitions, schedules, and execution history, gated by the `ManageReports` permission.

---

## Energy analytics

### Calculation modules

The energy system is built from six independent calculation modules that compose together:

**Consumption** -- Converts instantaneous power readings (kW) to energy (kWh) using trapezoidal integration over time-series samples.

**Demand** -- Calculates peak demand using a 15-minute sliding window. Produces a demand profile (peak per interval) and computes the load factor (average demand / peak demand).

**Degree days** -- Computes Heating Degree Days (HDD) and Cooling Degree Days (CDD) from outdoor temperature data. Runs linear regression of energy consumption against degree days and validates the model with CV-RMSE and NMBE statistics.

**Cost** -- Applies utility rate structures to consumption data. Supports four rate types:

| Rate type | Description |
|---|---|
| Flat | Single price per kWh |
| TOU (Time of Use) | Different prices by time period (on-peak, off-peak, shoulder) |
| Tiered | Price changes at consumption thresholds |
| Demand | Separate charge for peak demand ($/kW) in addition to energy charges |

**Rollup** -- Orchestrates daily and monthly aggregation. Runs consumption, demand, and cost calculations for each meter and writes the results as rollup records.

**Scheduler** -- Background task that runs the rollup module every 15 minutes. Uses upsert-on-conflict so partial rollups are updated in place as the day progresses, then finalized when the period ends.

### Storage

The `EnergyStore` persists to `data/energy.db` with four tables:

| Table | Contents |
|---|---|
| utility_rate | Rate definitions (type, prices, tiers, TOU periods) |
| energy_meter | Meter configurations (node bindings, rate assignments) |
| energy_baseline | Per-meter baseline data for degree-day regression |
| energy_rollup | Computed rollup records (daily and monthly, upsert on conflict) |

### GUI and API

The Energy section under Config has four sub-tabs:

- **Dashboard** -- Summary cards showing consumption, demand, and cost totals, plus a rollup table.
- **Meters** -- CRUD for meter configurations.
- **Rates** -- CRUD for utility rate definitions.
- **Baselines** -- Per-meter baseline list for degree-day analysis.

Ten API endpoints at `/api/energy/*` cover meter CRUD, rate CRUD, summary queries, consumption queries, and CSV export. All are gated by the `ManageEnergy` permission.

---

## Fault detection and diagnostics

### FDD engine

The `FddEngine` subscribes to the EventBus for value change events and also runs a 30-second periodic tick to evaluate time-dependent conditions. This dual approach catches both immediate threshold violations and slow-developing faults like stuck values.

### Condition types

| Condition | What it detects |
|---|---|
| Threshold | Value above or below a limit for a confirmation period |
| Stuck value | Value has not changed for longer than expected |
| Short-cycling | Equipment cycling on/off faster than a minimum run time |
| Schedule deviation | Actual value differs from scheduled setpoint beyond a tolerance |
| Demand-sensitive | Condition evaluated only during demand response periods or peak windows |

### Runtime state

Each rule-to-point binding maintains its own runtime state: confirmation counters track how long a condition has been true before raising a fault, `condition_since` timestamps record when the condition first appeared, and last-value fields enable stuck detection and short-cycling analysis.

### Events and integration

When a fault is confirmed, the engine publishes an `FddFaultRaised` event to the EventBus. When the condition clears, it publishes `FddFaultCleared`. These events flow through the same notification and webhook infrastructure as alarms -- any routing rule or webhook endpoint can react to FDD faults.

### Built-in rules

A set of default rules is seeded at startup covering common HVAC fault patterns. Operators can modify these or create new rules through the GUI.

### Storage and API

The `FddStore` persists to `data/fdd.db` with tables for rules, bindings (rule-to-point associations), faults (active and historical), and an execution log.

The GUI provides a FDD section under Config with three sub-tabs: Rules, Bindings, and Faults.

API endpoints at `/api/fdd/*` provide CRUD for rules and bindings, an auto-bind endpoint that matches rules to points based on tags, active and historical fault queries, and fault acknowledgment. All are gated by the `ManageFdd` permission.

---

## Export

### Architecture

The export system pushes operational data to external time-series databases and data warehouses. It is built on the `ExportConnector` trait, which defines a pluggable backend interface.

### Connectors

**InfluxDB** -- Ships by default. Uses the InfluxDB HTTP API to write history batches (`write_history_batch`) and alarm batches (`write_alarm_batch`).

**PostgreSQL** -- Available behind the `export-postgres` feature gate. Writes to a PostgreSQL database using structured tables.

### Event publishing

The `ExportPublisher` subscribes to the EventBus and forwards events to all configured connectors. Each connector has per-event-type toggles so operators can control exactly what gets exported:

- ValueChanged
- AlarmRaised
- AlarmCleared
- DeviceDown
- DeviceDiscovered

### Backfill

The backfill feature replays historical data from a specified start time through all configured connectors. This is useful when adding a new export target to a system that already has accumulated history. During backfill, the connector status shows as "backfilling" rather than "syncing."

### Status tracking

Each connector tracks its current state:

| Status | Meaning |
|---|---|
| idle | Configured but not actively exporting |
| syncing | Normal operation, forwarding live events |
| error | Last operation failed (details in status message) |
| backfilling | Replaying historical data |

### Storage and API

The `ExportStore` persists to `data/export.db` with tables for connector configurations and sync statuses.

The GUI provides an Export section under Config for managing connectors.

API endpoints at `/api/export/*` provide CRUD for connectors, a test endpoint to verify connectivity, a backfill trigger, and status queries. All are gated by the `ManageExport` permission.
