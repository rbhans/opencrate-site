# Outbound Connectors

## Notifications

Alarm routing can deliver through multiple channels, including webhook, email, and SMS flows. Those channels all sit behind a shared routing concept so operators configure recipients and rules rather than hard-coded per-feature destinations.

## Webhooks

Webhook delivery gives external systems a simple way to react to alarms, FDD events, and similar state changes. This keeps OpenCrate interoperable with lightweight automation platforms and custom integrations.

## MQTT

MQTT publishing is useful when you want OpenCrate to feed dashboards, data pipelines, or external automation tools. It turns the internal event stream into structured publishes against configurable topic patterns.

## Integration pattern

Across notifications, webhooks, and MQTT, the recurring design is:

1. detect or persist an important state change
2. emit an internal event
3. let specialized delivery modules fan that event outward
