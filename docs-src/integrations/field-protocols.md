# Field Protocols

## BACnet

BACnet support is one of the core reasons this project exists. The repository already includes support for BACnet/IP, BACnet/SC, and MS/TP related workflows, along with discovery, schedule sync, trend interactions, and other controller-facing features.

## Modbus

Modbus support complements BACnet for simpler devices and meters. The platform treats both protocols as inputs into the same higher-level node, history, alarm, and commissioning workflows.

## Why protocol abstraction matters

OpenCrate tries to keep protocol specifics close to the bridge and protocol modules while letting the rest of the application reason about points, devices, equipment, and events. That is what makes protocol-agnostic features like commissioning or dashboards possible.
