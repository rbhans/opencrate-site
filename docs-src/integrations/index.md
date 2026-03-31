# Integrations

## Field protocols

The codebase already includes first-class support for:

- BACnet/IP, BACnet/SC, and BACnet MS/TP related flows
- Modbus TCP and RTU related flows

Those integrations are organized so transport details stay close to the protocol layer while the rest of the platform works against more stable internal abstractions.

## External delivery and messaging

OpenCrate can also publish or deliver events outward through:

- MQTT
- webhooks
- email and SMS channels for notifications

## Design approach

The integration design is intentionally layered:

- field protocols bring data into the platform
- stores persist the important state
- event-driven delivery pushes meaningful changes back out

That separation is what lets new outbound integrations reuse existing alarm, history, and fault events instead of reinventing source logic.
