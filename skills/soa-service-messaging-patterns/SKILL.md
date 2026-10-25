---
name: soa-service-messaging-patterns
description: Design how services communicate at the SOA level using service messaging patterns (Thomas Erl / Arcitura) — Service Messaging, Messaging Metadata, State Messaging, Service Callback, Service Instance Routing, Asynchronous Queuing, Reliable Messaging, Event-Driven Messaging, Intermediate Routing, and Service Agent. Use this whenever the user is deciding between synchronous calls and messaging between services, needs to decouple services in time, guarantee delivery between services, notify consumers of events instead of having them poll, return results asynchronously via callbacks/webhooks, keep services stateless by carrying state in messages, route a conversation to a specific stateful service instance, put routing logic in intermediaries, or add event-driven sidecar/interceptor logic. Also trigger for service-mesh vs broker discussions and webhook design.
---

# SOA: Service Messaging Patterns

These patterns describe service-to-service communication at the **architecture level**: what each
interaction style gives you and what it costs. For channel-level and endpoint-level mechanics, see
the EIP skills. Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## The baseline

| Pattern | Distilled rule |
|---|---|
| [Service Messaging](https://patterns.arcitura.com/soa-patterns/design_patterns/service_messaging) | Services interact by **exchanging independent, self-contained messages** rather than through persistent, stateful connections or remote object references. That's the foundation of loose coupling. The cost is that each message must carry everything it needs, so reliability, correlation, and state need explicit patterns (below). |
| [Messaging Metadata](https://patterns.arcitura.com/soa-patterns/design_patterns/messaging_metadata) | Put **activity-specific metadata in the message** (headers): routing hints, correlation, security tokens, policies, and context, so services and intermediaries can process it without out-of-band agreements. More message processing overhead, and the header vocabulary needs governance. |

## Decoupling in time and reliability

| Pattern | Distilled rule |
|---|---|
| [Asynchronous Queuing](https://patterns.arcitura.com/soa-patterns/design_patterns/asynchronous_queuing) | Put a **queue between consumer and service**, so neither must be available at the same moment. The queue absorbs load spikes (load leveling) and the consumer doesn't block. The cost is that you can't get an immediate result, and you need correlation for any reply. |
| [Reliable Messaging](https://patterns.arcitura.com/soa-patterns/design_patterns/reliable_messaging) | Use an **intermediate reliability mechanism** (persistent broker, acknowledgements, retries, duplicate elimination) so messages arrive despite unreliable networks and runtimes. It adds latency and infrastructure. Realistically you get *at-least-once*, so receivers must be idempotent. |
| [Event-Driven Messaging](https://patterns.arcitura.com/soa-patterns/design_patterns/event_driven_messaging) | Consumers **subscribe** to events published by a service and are **notified when they happen**, instead of polling. That removes wasted polling and gives faster reaction. The cost is that publishers lose sight of who depends on them. Keep a catalog of events and subscribers, and treat event schemas as contracts. |
| [Service Callback](https://patterns.arcitura.com/soa-patterns/design_patterns/service_callback) | The consumer sends a request with a **callback address**, and the service **responds later by calling it** (webhook, reply queue). Good for long-running operations without blocking. The consumer must be reachable and handle late, duplicate, or missing callbacks. Sign callbacks (Data Origin Authentication) and correlate them to requests. |

**Choosing the interaction style:**
- The caller needs the answer right now to continue, and the operation is fast → synchronous request/response.
- The caller needs a result, but the work takes long → Service Callback (or async request + status polling).
- The caller just needs the work done eventually → Asynchronous Queuing + Reliable Messaging.
- Others need to know something happened → Event-Driven Messaging.

## State and instances

| Pattern | Distilled rule |
|---|---|
| [State Messaging](https://patterns.arcitura.com/soa-patterns/design_patterns/state_messaging) | **Carry state in the messages** instead of the service holding it between calls (like a continuation token or signed state blob). The service stays stateless and scales freely. The costs are bigger messages, and state must be protected from tampering (sign or encrypt it) and kept within message size limits. |
| [Service Instance Routing](https://patterns.arcitura.com/soa-patterns/design_patterns/service_instance_routing) | When consumers must keep talking to the **same stateful service instance**, the service returns an **instance identifier with its destination information in a standard format**. Consumers and infrastructure use it to route back to that instance, with no proprietary consumer logic. It may need infrastructure upgrades, and overuse makes messaging too stateful, against the Service Statelessness principle. Prefer State Repository or State Messaging when you can. Sticky sessions and session-ID routing are common examples. |

## Intermediaries

| Pattern | Distilled rule |
|---|---|
| [Intermediate Routing](https://patterns.arcitura.com/soa-patterns/design_patterns/intermediate_routing) | Move **dynamic routing decisions** (content-based, load-based, failover) **out of services into intermediary routing agents** (gateway, broker, service mesh). Services stay simpler and routing becomes configuration. It adds latency per hop, and routing failures are harder to debug. |
| [Service Agent](https://patterns.arcitura.com/soa-patterns/design_patterns/service_agent) | Move **event-driven logic that doesn't need explicit invocation** out of services and compositions into **service agents**: programs that intercept messages and **respond automatically to predefined conditions** (logging, auditing, validation, transformation, security checks). Compositions get smaller and faster. Agent logic is invisible in the contract, so document it. Agents can tie you to a vendor platform. Sidecars, mesh filters, and gateway policies are modern service agents. |

## Related skills

- `eip-messaging-channels-and-infrastructure`: Guaranteed Delivery, Point-to-Point vs Pub-Sub channels
- `eip-messaging-endpoints-and-consumers`: Idempotent Receiver and Transactional Client
- `eip-message-construction`: Return Address and Correlation Identifier for callbacks
- `soa-service-composition-patterns`: how messaging styles combine into compositions
