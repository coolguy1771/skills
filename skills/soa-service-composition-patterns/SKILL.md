---
name: soa-service-composition-patterns
description: Compose services into larger solutions using SOA composition patterns (Thomas Erl / Arcitura) — Capability Composition, Capability Recomposition, Agnostic Sub-Controller, Composition Autonomy, Atomic Service Transaction, and Compensating Service Transaction. Use this whenever the user is building a business process out of several services, asks "how do I make a multi-service operation atomic," "distributed transaction vs saga," "how do I undo partially completed work across services," "how do I reuse a piece of orchestration logic across processes," "why does my composition fail whenever any shared service is down," or is designing orchestrators/workflows (Temporal, Step Functions, Durable Functions, Camunda) across services.
---

# SOA: Service Composition Patterns

Composition is where service orientation pays off, because reusable capabilities get combined into
new solutions. It's also where distributed systems get hard: partial failures, consistency, and
shared dependencies. These patterns cover **how compositions are structured**, **how they stay
consistent**, and **how they stay available**. Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Structure

| Pattern | Distilled rule |
|---|---|
| [Capability Composition](https://patterns.arcitura.com/soa-patterns/design_patterns/capability_composition) | When a capability needs logic outside its service's boundary, it **composes capabilities of other services** instead of duplicating their logic. The composing service is the *composition controller*, and the others are *members*. Keep compositions shallow. Each extra level adds latency and new ways to fail. |
| [Capability Recomposition](https://patterns.arcitura.com/soa-patterns/design_patterns/capability_recomposition) | **Agnostic capabilities are deliberately reused in many compositions.** That's the goal of agnostic services and the main return on SOA investment. Recomposition increases a service's load and dependents, so heavily recomposed services need redundancy, capacity planning, and strict compatible-change discipline. |
| [Agnostic Sub-Controller](https://patterns.arcitura.com/soa-patterns/design_patterns/agnostic_sub_controller) | When part of a task service's composition logic turns out to be **reusable across processes** (it spans several entities but isn't tied to one parent process), extract it into a **sub-controller capability** that composes other services *and* can itself be composed. You get reuse of orchestration fragments, at the cost of more nesting. |

## Consistency across services

| Pattern | How it works | Choose it when | Costs |
|---|---|---|---|
| [Atomic Service Transaction](https://patterns.arcitura.com/soa-patterns/design_patterns/atomic_service_transaction) | Wrap the composition's runtime activities in a **transaction**: all commit or all roll back (2PC/XA, WS-AtomicTransaction) | Short-lived compositions, tightly controlled participants that all support the same transaction protocol, and a requirement for strict consistency | Locks held across services hurt availability and throughput. Rarely supported by cloud services and brokers. Couples participants at runtime |
| [Compensating Service Transaction](https://patterns.arcitura.com/soa-patterns/design_patterns/compensating_service_transaction) | Each step commits locally. On failure, run **compensating logic** that semantically undoes completed steps (refund, release reservation, cancel shipment). This is the saga pattern | Long-running or cross-organization processes, participants without shared transaction support, and most microservice systems | You must design a compensation for every step. Intermediate states are visible to others. Compensations can fail too, so they must be retryable and idempotent, with a human escalation path |

**Rule:** in modern distributed systems, **default to compensation (sagas)**. Use atomic
transactions only *within* a single service's database. When designing a saga:
- Order steps so the hardest to compensate (e.g., sending an email, charging a card) come **last**.
- Make every step and every compensation **idempotent**. They will be retried.
- Choose orchestration (a Process Manager or workflow engine) for complex sagas and choreography
  (events) for simple ones. See `eip-message-routing`.
- Make intermediate states explicit in the domain (`PENDING`, `RESERVED`) so other readers aren't
  misled.

## Availability of the whole composition

[Composition Autonomy](https://patterns.arcitura.com/soa-patterns/design_patterns/composition_autonomy):
a composition controller loses autonomy when its members, especially agnostic services shared with
other compositions, are busy or down. **Isolate the composition**: redundantly deploy the
composition's agnostic member services together with the task service in a dedicated environment.
That raises the autonomy and availability of the whole composition, at the cost of **more
infrastructure and governance** (several copies of the same service to patch and keep in sync).
Use it for business-critical compositions where shared-service contention or outages aren't
acceptable. In modern terms, it's a bulkhead or cell-based deployment.

Lighter-weight alternatives to full isolation: per-consumer bulkheads (separate connection pools
and quotas), circuit breakers and fallbacks for non-critical members, and asynchronous messaging to
members that don't need to answer immediately.

## Composition design checklist

1. Is the controller a task service (non-agnostic), with reusable fragments pulled into sub-controllers?
2. Is composition depth small, and is the end-to-end latency budget known?
3. Is the consistency model chosen explicitly: atomic only inside one service, compensation across services?
4. Does every step have an idempotent compensation, with failed compensations escalated?
5. Are critical members protected from shared contention (bulkheads, isolation, redundancy)?
6. Is the composition observable end to end (trace context through every member)?

## Related skills

- `eip-message-routing`: Process Manager and Routing Slip, the messaging mechanics of orchestration
- `soa-service-messaging-patterns`: interaction styles between composition members
- `soa-inventory-and-service-implementation`: Redundant Implementation, UI Mediator, State Repository
