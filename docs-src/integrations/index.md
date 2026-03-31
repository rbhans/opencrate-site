# Integrations

OpenCrate connects to building equipment through field protocols and pushes data outward through connectors. This section covers both sides.

## Design philosophy

Every integration in OpenCrate follows the same internal contract:

1. **Protocol drivers push raw values into the event bus.** A BACnet poll, a Modbus register read, or a COV notification all produce the same internal `ValueChanged` event. Nothing downstream knows or cares which wire the value arrived on.

2. **Outbound connectors subscribe to the event bus.** MQTT publishers, webhook dispatchers, and export connectors all receive the same stream of events and decide independently what to forward. Adding a new outbound path never requires changing protocol code.

3. **Stores are protocol-agnostic.** The node store, point store, alarm store, and history store hold normalized data. Protocol-specific details live in `ProtocolBinding` metadata on each node, not in the storage layer.

This separation means you can run a mixed BACnet/Modbus site and have every point appear identically in the GUI, the API, the alarm engine, and every outbound connector.

## Field protocols (inbound)

OpenCrate includes production-ready drivers for the two dominant building automation protocols:

| Protocol | Transports | Discovery | Write-back | Diagnostics |
|----------|-----------|-----------|------------|-------------|
| **BACnet** | IP, IPv6, Foreign/BBMD, BACnet/SC, MS/TP | Who-Is scan, object list enumeration | WriteProperty with priority array | Intrinsic reporting, event polling, alarm acknowledgment |
| **Modbus** | TCP, RTU (RS-485) | Configured device scan, TCP unit ID probe, RTU bus scan | Write with read-back verification | FC8 diagnostics, FC43 device identification |

Both protocols support multi-network configurations, per-device reconnection with exponential backoff, and parallel scanning. See [Field Protocols](field-protocols.md) for full details.

## Outbound connectors

Data flows out of OpenCrate through three paths:

| Connector | Purpose | Transport |
|-----------|---------|-----------|
| **MQTT** | Real-time telemetry streaming | MQTT 3.1.1 via rumqttc |
| **Webhooks** | Event notifications to external services | HTTP POST with retry |
| **Export** | Bulk data replication to time-series databases | InfluxDB HTTP API, PostgreSQL |

All three are EventBus subscribers. They start automatically during platform initialization and run for the lifetime of the process. See [Outbound Connectors](outbound-connectors.md) for configuration and behavior details.

## How integrations compose

A typical site might run:

- Two BACnet/IP networks and one Modbus RTU bus as field protocols
- MQTT publishing to a Grafana dashboard
- Slack webhooks for critical alarms
- InfluxDB export for long-term analytics

All of these run concurrently within a single OpenCrate process. The event bus handles fan-out: one `AlarmRaised` event triggers the MQTT publisher, the webhook dispatcher, and the export connector simultaneously, each applying its own filters and formatting independently.

## Adding new integrations

Protocol drivers implement the `ProtocolDriver` trait and push values through a `ValueSink`. Outbound connectors subscribe to the `EventBus` and filter for relevant event variants. Neither requires modifying existing code. See the [Developer Guide](../developer/index.md) for the full walkthrough.
