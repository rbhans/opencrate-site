# Automation and Analysis

## Logic engine

OpenCrate includes a visual logic system that compiles user-authored blocks into executable logic. The goal is to keep editing approachable while still running a real scriptable automation layer under the hood.

## Reporting

Reporting turns stored operational data into scheduled output. The reporting subsystem pulls from alarms, history, runtime summaries, and newer domain modules like energy.

Important traits of the current design:

- report definitions are persisted and schedulable
- rendering happens inside the application runtime
- delivery can reuse the same notification-oriented infrastructure patterns

## Energy

The energy module adds domain-specific structure on top of normal trend and schedule data:

- meters
- utility rates
- baseline records
- cost and demand summaries

This is a good example of how the platform can grow feature depth without changing the overall runtime model.

## FDD

Fault detection and diagnostics sits beside, not inside, the basic alarm engine. It evaluates rule logic against point data, tracks active faults, and emits events that other systems can react to.

That separation is useful because:

- standard alarms stay simple and immediate
- higher-order diagnostics can evolve independently
- outbound integrations can treat faults as another event stream
