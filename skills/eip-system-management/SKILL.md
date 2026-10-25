---
name: eip-system-management
description: Operate, observe, test, and debug messaging systems using the Enterprise Integration Patterns for system management — Control Bus, Detour, Wire Tap, Message History, Message Store, Smart Proxy, Test Message, and Channel Purger. Use this whenever the user asks how to monitor or administer a distributed messaging system, tap into a queue to see traffic without disturbing consumers, trace a message's path through many hops, audit or report on messages, measure request-reply latency, inject synthetic/canary messages to detect garbled output, temporarily route traffic through validation or debugging, or clean leftover messages out of a channel between tests or after an incident. Also trigger for messaging observability, tracing (OpenTelemetry) across brokers, and integration-test hygiene.
---

# EIP: System Management

Loose coupling has an operational cost: no single component can see the whole flow. These patterns
restore visibility and control **without reintroducing coupling**. The common move is to add
management behaviour *around* channels and components instead of inside the business logic.

## Control

### [Control Bus](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ControlBus.html)
**Problem:** administer a messaging system spread across platforms and regions.
**Solution:** use the messaging infrastructure itself for management. Give each component a
**separate control channel** for configuration commands, heartbeats, statistics, and exceptions.
**Rules:** keep control traffic **separate from data channels**, so a backlog of data never
blocks a "pause" or "reconfigure" command. Secure it more tightly than the data plane, since
whoever controls the bus controls every component. Today this is often a mix of a control plane
(Kubernetes operators, admin APIs) and telemetry pipelines, but the principle is the same.

### [Detour](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Detour.html)
**Problem:** temporarily route messages through extra steps (validation, testing, debugging).
**Solution:** a **context-based router** controlled by the Control Bus. In one state it sends
messages through the extra steps. In the other it passes them straight through.
**Uses:** turning on deep validation during an incident, mirroring traffic into a debugger, adding
a canary step. Make the detour switchable at runtime and default it to **off**, so the extra
latency isn't paid all the time.

## Observation

| Pattern | Distilled rule |
|---|---|
| [Wire Tap](https://www.enterpriseintegrationpatterns.com/patterns/messaging/WireTap.html) | Inspect a **point-to-point** channel without stealing messages from the real consumer. Put a simple *recipient list* on the channel that sends each message to the original output **and** a secondary inspection channel. Use it for auditing, analytics, and debugging. Never let the secondary path slow down or fail the primary path. Watch out for sensitive data copied onto the tap. |
| [Message History](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageHistory.html) | Each component that handles a message **appends its identity** to a history list in the message. Debugging can then reconstruct the path, and loops can be detected (a component that sees itself in the history drops or flags the message). Distributed tracing (W3C trace context + spans) does the same job out of band. Prefer it when available, and keep a compact in-message history only where you need loop detection. |
| [Message Store](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageStore.html) | Report on message data without disturbing the transient messaging system: **wire-tap a copy of each message into a central store** (often only key headers and fields). That supports audit, SLA reporting, and "where is my order?" queries. Decide retention, PII handling, and what level of detail to store before you turn it on. A store with full bodies grows fast and becomes a compliance liability. |
| [Smart Proxy](https://www.enterpriseintegrationpatterns.com/patterns/messaging/SmartProxy.html) | Track a service that replies to whatever **Return Address** the requestor supplied. The proxy intercepts the request, **saves the original return address** keyed by correlation ID, **substitutes its own reply channel**, and forwards the reply to the original requestor once it has measured or logged it. It's useful for latency metrics and debugging request-reply services you can't modify. The proxy keeps state, so it needs a timeout for replies that never arrive. |

## Testing and hygiene

### [Test Message](https://www.enterpriseintegrationpatterns.com/patterns/messaging/TestMessage.html)
**Problem:** a component is still processing messages (so health checks pass), but an internal
fault garbles its output.
**Solution:** a **test data generator** injects known test messages into the live stream. A
**test message separator** pulls the results out and a **verifier** compares them with expected
output.
**Rules:** mark test messages clearly (a header) so business systems never act on them. Synthetic
transactions in production (canaries) are the modern form. Alert when verification fails, not
just when the component is down. This catches the "running but wrong" failure that liveness
probes miss.

### [Channel Purger](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ChannelPurger.html)
**Problem:** leftover messages on a channel disturb tests or running systems.
**Solution:** a tool that **removes unwanted messages** from a channel, either all of them or by
selector.
**Rules:** use it routinely in integration-test setup and teardown, and isolate test channels per
run (a unique queue name per test) where you can. In production, **never purge blindly**. Move
the messages to a holding channel or Message Store first, because purged data can't be recovered
and may be someone's in-flight business transaction.

## A minimal observability baseline for any messaging system

1. Queue depth, age of the oldest message, and consumer lag per channel, with alerts.
2. Dead-letter and invalid-message channel counts, with an owner and a replay runbook.
3. End-to-end trace propagation through headers (Message History / trace context).
4. A wire-tapped Message Store of key metadata for audit and lookups.
5. Synthetic Test Messages checking correctness, not just liveness.
6. A Control Bus or admin API to pause, resume, and reconfigure consumers without redeploying.

## Related skills

- `eip-message-routing`: the routers that Detour and Wire Tap are built from
- `eip-message-construction`: Correlation Identifier and Return Address, which Smart Proxy relies on
- `observability-platform-standardization`: platform-wide telemetry standards
