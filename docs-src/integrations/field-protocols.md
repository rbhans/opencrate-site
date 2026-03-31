# Field Protocols

## BACnet

BACnet support is built on the `rustbac` family of crates (`rustbac-core`, `rustbac-client`, `rustbac-datalink`, `rustbac-bacnet-sc`, `rustbac-mstp`) and covers the full range of BACnet services needed for a production building automation system.

### Transport modes

OpenCrate supports five BACnet transport modes, all managed through a unified `TransportClient` enum that wraps `Arc<BacnetClient<T>>`. The `with_client!` macro dispatches calls to the correct transport without generic proliferation in calling code.

| Mode | Config key | Description |
|------|-----------|-------------|
| **IP** | `mode: "ip"` | Standard BACnet/IP unicast and broadcast on UDP port 47808. The default for most installations. |
| **IPv6** | `mode: "ipv6"` | BACnet/IP over IPv6 networks. |
| **Foreign (BBMD)** | `mode: "foreign"` | Registers with a BACnet Broadcast Management Device for cross-subnet communication. Requires `bbmd_addr` and `ttl` in config. Uses `BacnetClient::new_foreign()`. |
| **BACnet/SC** | `mode: "secure_connect"` | BACnet Secure Connect over WebSocket with TLS. Requires `hub_endpoint`. Uses `BacnetClient::new_sc()`. |
| **MS/TP** | `mode: "mstp"` | RS-485 serial via the `rustbac-mstp` crate. Implements the MS/TP data link layer with CRC-8 (header) and CRC-16 (data) validation, frame encode/decode, and preamble scanning. Requires `serial_port`, `baud_rate`, `mac_address`, and `max_master`. |

### Multi-network support

`BacnetNetworks` manages multiple concurrent `BacnetBridge` instances, each keyed by a `network_id` string (e.g., `"ip-main"`, `"mstp-field"`). This allows a single OpenCrate instance to communicate across segmented BACnet networks simultaneously.

Configuration uses `ScenarioSettings.bacnet_networks`, a map of network IDs to `BacnetNetworkConfig` objects. Legacy configurations with a single `bacnet` field are automatically wrapped as `{"default": config}` through `resolved_bacnet_networks()`.

```json
{
  "bacnet_networks": {
    "ip-main": {
      "mode": "ip",
      "monitor_interval_secs": 300,
      "object_check_cycles": 6,
      "trend_log_sync_interval_secs": 600
    },
    "mstp-field": {
      "mode": "mstp",
      "serial_port": "/dev/ttyUSB0",
      "baud_rate": 76800,
      "mac_address": 1,
      "max_master": 127
    }
  }
}
```

**`BacnetNetworkConfig` fields:**

| Field | Default | Description |
|-------|---------|-------------|
| `mode` | `"ip"` | Transport mode (ip, ipv6, foreign, secure_connect, mstp) |
| `bbmd_addr` | -- | BBMD address for foreign mode |
| `ttl` | -- | Time-to-live for BBMD registration |
| `hub_endpoint` | -- | WebSocket endpoint for BACnet/SC |
| `server_device_instance` | -- | Device instance for the local BACnet server |
| `serial_port` | -- | Serial port path for MS/TP |
| `baud_rate` | -- | Baud rate for MS/TP |
| `mac_address` | -- | MS/TP MAC address (0-127) |
| `max_master` | -- | Highest MS/TP master address to poll |
| `monitor_interval_secs` | 300 | Interval for background device monitoring |
| `object_check_cycles` | 6 | Monitor cycles between object list change checks |
| `trend_log_sync_interval_secs` | 600 | Interval for trend log backfill sync |

### Discovery

Discovery is always user-initiated and never runs automatically on startup. Re-discovery preserves existing device state (accepted devices stay accepted).

**Scan process:**

1. OpenCrate broadcasts Who-Is on the selected network(s).
2. Responding devices are recorded as `DiscoveredDevice` entries with protocol-agnostic fields plus `protocol_meta` JSON carrying BACnet-specific data.
3. For each device, OpenCrate reads the full object list and creates `DiscoveredPoint` entries.
4. Device metadata is enriched by reading: VendorName, ModelName, FirmwareRevision, Location, Description, MaxApduLengthAccepted, SegmentationSupported, ProtocolVersion, and ApplicationSoftwareVersion.

**Parallel multi-network scan:** `scan_bacnet_all()` scans all configured networks concurrently using `join_all`. Single-network configurations skip the parallelism overhead.

**Who-Has discovery:** `who_has_by_id()` and `who_has_by_name()` provide broadcast discovery of specific objects by identifier or name string.

**Accepting devices:** When a discovered device is accepted, OpenCrate creates nodes and entities with auto-generated tags in a single pass. Each discovered point includes `network_id` in its `protocol_meta`, enabling write routing to the correct network.

### Device monitoring

A background monitor loop runs continuously after startup:

- **Who-Is polling:** Sends Who-Is to all known devices at a configurable interval (default 300 seconds).
- **Offline detection:** A device that misses two consecutive poll cycles triggers a `DeviceDown` event.
- **Recovery detection:** When an offline device responds again, it is automatically recovered.
- **Passive discovery:** New devices that respond to Who-Is broadcasts but were not previously known are recorded.
- **Object list change detection:** Every N monitor cycles (default 6, approximately every 30 minutes), the monitor reads the ObjectList length for each device. A mismatch publishes an `ObjectListChanged` event, and the discovery store marks the device as `object_list_stale`.

The monitor publishes a `DeviceMonitorCycle` summary event after each complete cycle.

### Reading values

**Poll loop:** Reads PresentValue and StatusFlags together for each monitored point. StatusFlags (IN_ALARM, FAULT, OVERRIDDEN, OUT_OF_SERVICE) are mapped to internal `PointStatusFlags`.

**Change of Value (COV):** Supports both `SubscribeCOV` and `SubscribeCOVProperty` for granular property-level subscriptions. COV notifications are processed through the same StatusFlags mapping. Subscriptions can be cancelled with `cancel_cov_property_subscription()`.

**Priority array:** `read_priority_array(device_instance, object_id)` reads all 16 priority levels plus the RelinquishDefault, returning a `PriorityArrayInfo` structure. The GUI provides a dropdown for selecting write priority (1-16) with standard BACnet priority labels.

### Writing values

**Network-aware routing:** The write handler extracts the BACnet device instance via `extract_bacnet_instance()`, then calls `find_network_for_device()` to route the write to the correct `BacnetBridge`. If direct routing fails, it iterates all networks as a fallback.

**Write priority:** `PointSource::write_point` accepts an optional `priority: Option<u8>` parameter (1-16). The GUI exposes this as a priority dropdown with BACnet-standard labels.

### Trend logs

`read_trend_log` reads trend log entries from a device. `backfill_trend_log` replays historical data from a device's trend log buffer into the local history store.

The sync interval is configurable per network (default 600 seconds). `reset_trend_log_backfill()` restarts the sync loop with fresh state. The discovery UI provides a per-entry Backfill button for manual triggering.

### Schedule interop

Full read/write support for BACnet schedule objects:

| Function | Description |
|----------|-------------|
| `read_schedule` | Read a schedule object's weekly schedule |
| `write_schedule` | Write a weekly schedule to a device |
| `read_schedule_default` | Read the schedule's default value |
| `read_calendar` | Read a calendar object's date list |
| `read_exception_schedule` | Read exception schedule entries |

The schedule view includes a BACnet Sync panel with Read/Write from Device buttons for bidirectional synchronization.

### Intrinsic reporting and alarms

**Event polling:** A background loop checks `recv_event_notification` and calls `GetEventInformation` for each device every 60 seconds. Events are enriched with `acknowledged_transitions`, `notify_type`, `event_enable`, and `event_priorities`.

**Alarm acknowledgment:** `acknowledge_alarm()` sends the AcknowledgeAlarm service to the remote device. The GUI Ack button automatically sends a BACnet acknowledgment for devices with `bacnet-` prefixed IDs.

**Alarm/event queries:** `get_alarm_summary()` and `get_enrollment_summary()` are also exposed for bulk alarm state retrieval.

### Device management

| Function | Description |
|----------|-------------|
| `reinitialize_device` | Warmstart or coldstart a remote device (split buttons in UI) |
| `device_communication_control` | Enable or disable communication on a device |
| `sync_time` | Send UTC time synchronization (also runs automatically every 4 hours) |
| `get_event_info` | Retrieve event information (shown in Events table in discovery UI) |

### Object management

| Function | Description |
|----------|-------------|
| `create_object` | Create a new BACnet object on a remote device |
| `delete_object` | Delete a BACnet object from a remote device |

The commissioning UI in the discovery view provides a type selector for object creation.

### File transfer

| Function | Description |
|----------|-------------|
| `read_file_stream` | Stream-read a file from a device |
| `read_file_record` | Record-read a file from a device |
| `write_file_stream` | Stream-write a file to a device |
| `write_file_record` | Record-write a file to a device |

A File Transfer UI panel is available in the discovery view for accepted devices.

### Private transfer

`private_transfer()` sends vendor-specific PrivateTransfer requests for integrations that extend beyond the standard BACnet service set.

### BACnet server

OpenCrate can act as a BACnet server, exposing local points to other BACnet devices on the network:

- **Service handlers:** WritePropertyMultiple, SubscribeCOV, CreateObject, DeleteObject
- **COV subscription manager:** Tracks subscriptions with lifetime and expiry. Generates `UnconfirmedCOVNotification` messages when subscribed properties change.
- **Object store:** Populated from the local PointStore via `init_server_store()`.
- **Configuration:** `server_device_instance` in `BacnetNetworkConfig` sets the device instance number for the local server.

### Reconnection

Per-device exponential backoff with a 2-second base and 5-minute maximum. After 5 consecutive failures, a `DeviceDown` event is published. Devices automatically recover when communication is restored.

---

## Modbus

Modbus support is built on the `rustmod` family of crates (`rustmod-core`, `rustmod-client`, `rustmod-datalink`) and provides TCP and RTU transport with full function code coverage.

### Transport modes

`ModbusTransport` is an enum wrapping TCP and RTU variants, dispatching all read/write methods uniformly.

| Mode | Description |
|------|-------------|
| **TCP** | One persistent connection per device, stored in `clients: HashMap<String, Arc<ModbusTransport>>`. Writes reuse existing connections. |
| **RTU** | Shared serial transport keyed by port via the `rustmod-datalink` `rtu` feature. All RTU devices on the same bus share one serial connection. Configured with `serial_port`, `baud_rate`. |

Configuration through `ModbusNetworkConfig`:

```json
{
  "modbus": {
    "mode": "tcp",
    "default_timeout_ms": 1000,
    "default_retry_count": 3
  }
}
```

For RTU:

```json
{
  "modbus": {
    "mode": "rtu",
    "serial_port": "/dev/ttyUSB1",
    "baud_rate": 9600
  }
}
```

### Per-device client configuration

Each device can override defaults through `ModbusDefaults`:

| Field | Description |
|-------|-------------|
| `response_timeout_ms` | Maximum wait time for a device response |
| `retry_count` | Number of retries before marking a read as failed |
| `throttle_delay_ms` | Minimum delay between consecutive requests to a device |

### Block reads

`plan_block_reads()` coalesces contiguous or nearby register addresses into block reads to minimize network round-trips:

- **Maximum gap:** 10 registers. Addresses within this gap are read as a single block even if not all are mapped.
- **Block size cap:** 125 registers (holding/input registers) or 2000 bits (coils/discrete inputs), matching Modbus protocol limits.
- **Result:** One read per block instead of N individual reads. Points are extracted from the block response by offset.

### Bit extraction

Individual bits or bit fields within a register are accessible through `ModbusPointMapping` fields:

- `bit_offset: Option<u8>` -- Extract a single bit at this position.
- `bit_mask: Option<u16>` -- Extract a masked bit field from the register.

`apply_bit_extraction()` handles the extraction. When writing to a point with `bit_mask`, FC22 (Mask Write Register) is used automatically to modify only the relevant bits without disturbing other bits in the register.

### Function code support

| FC | Function | Description |
|----|----------|-------------|
| FC1-4 | Standard reads | Read coils, discrete inputs, holding registers, input registers |
| FC5-6, FC15-16 | Standard writes | Write single/multiple coils and registers |
| FC8 | Diagnostics | Echo test, bus message counters, clear counters. Exposed through `diagnostics(device_id, sub_function, data)`. |
| FC22 | Mask Write Register | Atomic bit-level register modification. Used automatically for points with `bit_mask`. Exposed through `mask_write_register(device_id, addr, and_mask, or_mask)`. |
| FC23 | Read/Write Multiple | Atomic read and write in a single transaction via `read_write_multiple()`. |
| FC24 | Read FIFO Queue | Queue read via `read_fifo(device_id, addr)`. |
| FC43 | Read Device Identification | Returns `DeviceIdInfo { vendor, product, revision }`. Used during discovery enrichment. |

### Register browser

`read_registers()` and `read_bits()` support arbitrary register reads from the GUI, independent of configured point mappings. This is useful for commissioning and troubleshooting. The Modbus Registers tab in the discovery view provides the interface.

### Discovery

Modbus devices do not self-describe their register maps, so discovery focuses on device presence detection and identity enrichment:

**Configured device scan:** Reads all devices defined in the scenario configuration and verifies they respond.

**TCP unit ID probe:** `scan_unit_ids(host, port, start, end)` probes a range of unit IDs with a test read (1-second timeout, no retries). Each responding unit ID is checked with FC43 for vendor/model/firmware information.

**RTU bus scan:** `scan_rtu_unit_ids(start, end)` performs the same probe over the existing RTU serial bus. `is_rtu()` exposes the bridge mode so the GUI can adapt the scan form (TCP vs RTU).

**Discovery UI adaptation:** The collapsible scan form adapts based on bridge mode. Protocol badges are color-coded (blue for BACnet, amber for Modbus). The Modbus overview tab explains that Modbus does not self-describe registers and directs users to profiles and the Registers tab.

### Profile library

Pre-built device profiles in `profiles/modbus-library/` provide register maps for common devices:

- `generic-power-meter` -- Generic power meter template
- `schneider-pm5300` -- Schneider Electric PM5300 power meter
- `abb-acs580` -- ABB ACS580 variable frequency drive

`load_profile_library(dir)` loads all JSON profiles from the directory. `match_profile(profile, vendor, model)` scores each profile 0.0-1.0 by manufacturer and model string similarity. `DiscoveryService::apply_modbus_profile()` converts matched profile points into `DiscoveredPoint` entries with appropriate Modbus bindings.

Custom profiles can be added by placing JSON files in the profile library directory.

### Write-back verification

After every write operation, OpenCrate reads the register back to confirm the device accepted the value. If the read-back fails (device busy, timeout), the system falls back to using the written value and logs the discrepancy.

### COV filtering

`PointStore::set_if_changed()` skips event publication and history recording when a polled value is unchanged from the previous read. This reduces event bus noise and history storage for Modbus points that are polled on a fixed interval.

### Error handling

`ModbusError` is a structured enum that preserves the full context of failures:

| Variant | Description |
|---------|-------------|
| `Exception` | Device responded with a Modbus exception code (preserved as `ExceptionCode`) |
| `Transport` | Network-level failure |
| `Timeout` | Response not received within deadline |
| `InvalidResponse` | Response did not match expected format |
| `Encode` | Request encoding failure |

Helper methods `is_busy()` and `is_transport()` support retry logic. The poll loop logs the error category for each failure.

**Short block response:** If a device returns fewer registers than requested, affected points are logged with a warning and marked with FAULT status.

### Reconnection

Uses the same `DeviceBackoff` implementation as BACnet (shared in `src/bridge/backoff.rs`): exponential backoff with a 2-second base and 5-minute maximum. After 5 consecutive failures, a `DeviceDown` event is published through the EventBus. Devices recover automatically when communication resumes.

### GUI components

Two dedicated tabs appear in the discovery view for accepted Modbus devices:

- **ModbusRegisters** (`modbus_device_registers.rs`): Register browser for arbitrary reads, FC43 device identification display, and FC24 FIFO queue reads.
- **ModbusDiagnostics** (`modbus_device_diagnostics.rs`): FC8 echo test and bus counters, FC22 mask write interface.
