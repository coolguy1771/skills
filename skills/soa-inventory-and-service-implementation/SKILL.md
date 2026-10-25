---
name: soa-inventory-and-service-implementation
description: Implement services and their supporting inventory infrastructure using SOA patterns (Thomas Erl / Arcitura) — foundational service patterns Functional Decomposition, Service Encapsulation, Agnostic Context, Non-Agnostic Context, Agnostic Capability; inventory implementation patterns Canonical Resources, Cross-Domain Utility Layer, Dual Protocols, Augmented Protocols, Inventory Endpoint, State Repository, Stateful Services, Service Grid; and service implementation patterns Service Façade, Partial State Deferral, Partial Validation, Redundant Implementation, Service Data Replication, Reference Data Centralization, UI Mediator, Microservice Deployment, Containerization. Use this whenever the user is deciding what logic becomes a service and what stays a task service, how to keep a service's contract stable while its internals change, how to scale or make stateful services highly available, whether a service should own a replica of its data, how to expose an inventory to external consumers, how to support more than one protocol, how to give users progress feedback during long compositions, or how to deploy services as autonomous microservices and containers.
---

# SOA: Service and Inventory Implementation

These patterns move from *which services exist* (`soa-service-inventory-design`) to **how each
service is shaped, built, deployed, and kept autonomous**. They come back to two forces again and
again: **autonomy** (a service controls its own runtime and data) and **statelessness** (a service
holds as little state as possible between calls). Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Foundational service patterns: from problem to service

Apply these in order. They're the basic service-oriented analysis steps.

1. **[Functional Decomposition](https://patterns.arcitura.com/soa-patterns/design_patterns/functional_decomposition)**:
   break a large business problem into smaller units of logic that can be solved separately.
2. **[Service Encapsulation](https://patterns.arcitura.com/soa-patterns/design_patterns/service_encapsulation)**:
   decide which of those units are worth exposing as **services**, meaning they have reuse or
   composition potential. Not everything should be a service.
3. **[Agnostic Context](https://patterns.arcitura.com/soa-patterns/design_patterns/agnostic_context)**:
   put **multipurpose logic** (useful to many processes) in its own **agnostic services**
   (entity/utility).
4. **[Non-Agnostic Context](https://patterns.arcitura.com/soa-patterns/design_patterns/non_agnostic_context)**:
   put **single-purpose process logic** in **task services**. It isn't reusable, but it's still
   valuable as a governed service in the inventory.
5. **[Agnostic Capability](https://patterns.arcitura.com/soa-patterns/design_patterns/agnostic_capability)**:
   inside agnostic services, define capabilities **generically** (e.g., `GetCustomer`, not
   `GetCustomerForInvoiceRun`), so many consumers can use them.

**Trap:** agnostic services that gradually gain process-specific capabilities. When a
capability's name mentions a specific consumer or process, it belongs in a task service.

## Shaping one service internally

| Pattern | Distilled rule |
|---|---|
| [Service Façade](https://patterns.arcitura.com/soa-patterns/design_patterns/service_facade) | Put a **façade component between the contract and the core logic**. Changes to the core don't ripple into the contract, the same core can sit behind several contracts (Concurrent Contracts), and you have a place for contract-specific behaviour. It's a hexagonal-architecture adapter under another name. |
| [Partial Validation](https://patterns.arcitura.com/soa-patterns/design_patterns/partial_validation) | **Consumers** validate only the parts of a message they actually use and ignore the rest. That reduces coupling to the full contract and saves processing. It's the tolerant reader principle. It doesn't relax the *provider's* own input validation. |
| [Partial State Deferral](https://patterns.arcitura.com/soa-patterns/design_patterns/partial_state_deferral) | While a service waits in the middle of a long activity, **move part of its state out of memory** (to a state repository or the message) to save memory. Accept the cost of saving and reloading it. |
| [Reference Data Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/reference_data_centralization) | Reference data (codes, lookup lists) used by many services is **stored and managed centrally** and reached through a dedicated reference-data utility service. That removes redundant copies and governance drift. Cache the lookups so the extra hop doesn't add latency and risk. |
| [UI Mediator](https://patterns.arcitura.com/soa-patterns/design_patterns/ui_mediator) | When a composition runs a long time, a **mediator service gives the user interface immediate, incremental feedback** (progress, partial results) instead of a blocked screen. Examples: async job + status endpoint, WebSocket/SSE progress, optimistic UI. |

## State and scale

| Pattern | Distilled rule |
|---|---|
| [State Repository](https://patterns.arcitura.com/soa-patterns/design_patterns/state_repository) | During long-running activities, **defer state to a dedicated repository** (DB, cache) instead of keeping it in memory, so services can stay stateless and scale. Durable workflow engines persist state this way. |
| [Stateful Services](https://patterns.arcitura.com/soa-patterns/design_patterns/stateful_services) | Delegate state management to **dedicated stateful utility services**, so the other services stay stateless. Those stateful services become critical infrastructure and must be highly available. |
| [Service Grid](https://patterns.arcitura.com/soa-patterns/design_patterns/service_grid) | Defer state to a **grid of replicated, distributed cache/state nodes**. If one node fails, state survives on another, which gives high availability and horizontal scale. Examples: distributed caches and data grids like Hazelcast, Infinispan, Redis Cluster. It adds infrastructure and costs consistency trade-offs. |
| [Redundant Implementation](https://patterns.arcitura.com/soa-patterns/design_patterns/redundant_implementation) | **Deploy several copies of a service** (often a heavily reused agnostic one) behind load balancing or failover, so a high-demand service isn't a single point of failure. Today that means replicas, zones, and regions. Pair it with stateless design or one of the state patterns above. |
| [Service Data Replication](https://patterns.arcitura.com/soa-patterns/design_patterns/service_data_replication) | Give a service its **own replicated copy of the data it needs**, so it doesn't contend for a shared database. That's more autonomy and performance at the cost of replication lag and eventual consistency. CDC-fed read models are a modern form. |

## Inventory-wide implementation

| Pattern | Distilled rule |
|---|---|
| [Canonical Resources](https://patterns.arcitura.com/soa-patterns/design_patterns/canonical_resources) | Standardize the **infrastructure resources** services use (runtime platforms, databases, messaging, observability stack) through enterprise design standards, so the estate doesn't grow a different stack per service. Golden paths and paved roads are the modern form. Too much shared infrastructure reduces autonomy, so standardize the *kind* of resource, not necessarily shared *instances*. |
| [Cross-Domain Utility Layer](https://patterns.arcitura.com/soa-patterns/design_patterns/cross_domain_utility_layer) | Utility logic duplicated across domain inventories is pulled into a **common utility layer that spans the domains** (notification, audit, document generation). It recovers reuse that domain inventories would otherwise lose, but needs cross-domain governance. |
| [Dual Protocols](https://patterns.arcitura.com/soa-patterns/design_patterns/dual_protocols) | Keep a **primary** canonical protocol for interoperability and allow a **secondary** protocol where performance requires it (e.g., HTTP+JSON for most services, gRPC between hot-path internal services). More governance, and services must support both where required. |
| [Augmented Protocols](https://patterns.arcitura.com/soa-patterns/design_patterns/augmented_protocols) | Keep the canonical protocol as the baseline, and **add more beneficial protocols** on services where it's feasible and useful (streaming, binary, async). Consumers get better service levels without losing the canonical baseline. It's the newer, more flexible version of Dual Protocols. |
| [Inventory Endpoint](https://patterns.arcitura.com/soa-patterns/design_patterns/inventory_endpoint) | When **external consumers** need some of an inventory's capabilities, provide a **dedicated endpoint service** as their single official entry point. It exposes only what they need and can add policies for them, so the underlying services keep their freedom to change. The cost is duplicated contracts to maintain. Similar to an edge API/gateway product for a domain. |

## Deployment

| Pattern | Distilled rule |
|---|---|
| [Microservice Deployment](https://patterns.arcitura.com/soa-patterns/design_patterns/microservice_deployment) | Deploy each microservice (usually from the Micro Task Abstraction layer) in its **own autonomous runtime**, with independent deployment, scaling, and release cadence. It increases infrastructure and governance overhead. |
| [Containerization](https://patterns.arcitura.com/soa-patterns/design_patterns/containerization) | Package each service with its system dependencies into an **independently manageable container image**, deployed and run by a container engine or orchestrator. It reduces footprint compared with full VMs, isolates failures, and speeds recovery and scaling. It adds infrastructure (registry, orchestrator) and administrative overhead. |

## Related skills

- `soa-service-inventory-design`: the layers and standards these implementations live within
- `soa-service-messaging-patterns`: State Messaging, another way to keep services stateless
- `containerize-application`, `setup-kubernetes-deployment`: practical containerization
