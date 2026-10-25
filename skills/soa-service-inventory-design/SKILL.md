---
name: soa-service-inventory-design
description: Design and govern a service inventory using SOA patterns (Thomas Erl / Arcitura) — foundational inventory patterns Enterprise Inventory, Domain Inventory, Service Normalization, Logic Centralization, Service Layers, Canonical Protocol, Canonical Schema; logical layer patterns Entity Abstraction, Process Abstraction, Utility Abstraction, Micro Task Abstraction; centralization patterns Policy, Process, Rules, and Schema Centralization; and governance patterns Canonical Expression, Canonical Versioning, Metadata Centralization. Use this whenever the user is planning a portfolio of services or microservices across a domain or enterprise, asks "how do we stop teams building duplicate services," "how should services be layered (entity/task/utility)," "one enterprise-wide service catalog or per-domain," "where should business rules and policies live," "should we standardize on one protocol and schema," or needs naming and versioning standards and a service registry. Also trigger for platform-engineering service catalogs and domain decomposition at enterprise scale.
---

# SOA: Service Inventory Design and Governance

A **service inventory** is an independently standardized and governed collection of services. These
patterns decide the inventory's **scope**, its **boundaries**, its **layers**, and **what is
standardized** across it. They apply directly to modern microservice estates and internal
developer platforms, not just WS-* era SOA. Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Scope: how big is one inventory?

| Pattern | Distilled rule |
|---|---|
| [Enterprise Inventory](https://patterns.arcitura.com/soa-patterns/design_patterns/enterprise_inventory) | One inventory for the whole enterprise, with one set of standards, which maximizes reuse and recomposition across the organization. Works only with strong central governance and an organization that will actually agree on shared standards. |
| [Domain Inventory](https://patterns.arcitura.com/soa-patterns/design_patterns/domain_inventory) | Several inventories, each **standardized and governed separately within a meaningful domain boundary** (business unit, bounded context). The realistic choice for large or federated organizations. The cost is transformation at domain boundaries (see `soa-legacy-encapsulation-and-transformation`), so choose boundaries where cross-domain traffic is low. |

Default to **domain inventories** unless the enterprise is small or tightly governed. A global
standard that nobody follows is worse than domain standards that everyone follows.

## Boundaries: avoid duplicate logic

| Pattern | Distilled rule |
|---|---|
| [Service Normalization](https://patterns.arcitura.com/soa-patterns/design_patterns/service_normalization) | Service boundaries **must not overlap**. Each capability exists in exactly one service. Model the inventory blueprint *before* building, so overlap is caught on paper. |
| [Logic Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/logic_centralization) | Consumers **must use the official service** for a given kind of logic, never their own copy, and never reach past the service into its database. Normalization designs out duplication. Centralization keeps it out over time, which requires governance (catalog, reviews, discoverability). |

## Layers: classify services by the logic they hold

[Service Layers](https://patterns.arcitura.com/soa-patterns/design_patterns/service_layers) organize
the inventory into logical layers by **type of logic** and **reusability**:

| Layer pattern | Contains | Reuse | Modern analogue |
|---|---|---|---|
| [Utility Abstraction](https://patterns.arcitura.com/soa-patterns/design_patterns/utility_abstraction) | Cross-cutting, **non-business-centric** logic: logging, notification, document generation, currency conversion | High, across domains | Platform/shared services |
| [Entity Abstraction](https://patterns.arcitura.com/soa-patterns/design_patterns/entity_abstraction) | **Business-entity-centric**, agnostic logic: Customer, Invoice, Product | High, within a domain | Domain/aggregate services, System APIs |
| [Process Abstraction](https://patterns.arcitura.com/soa-patterns/design_patterns/process_abstraction) | **Non-agnostic** logic for one parent business process. These are *task services* that compose entity and utility services | Low, by design | Process APIs, orchestrators, BFFs |
| [Micro Task Abstraction](https://patterns.arcitura.com/soa-patterns/design_patterns/micro_task_abstraction) | Small units of non-agnostic logic with **specialized processing or deployment needs** (extreme performance, scaling, availability), split out of task services into a microservice layer | Low | Microservices deployed independently (usually with Microservice Deployment + Containerization) |

**Rules:** agnostic (reusable) logic and non-agnostic (single-purpose) logic belong in different
services. Mixing them is the most common reason "reusable" services aren't reused. Micro Task
Abstraction adds analysis, governance, and infrastructure overhead, so use it only where the
specialized requirements are real.

## What to standardize across the inventory

| Pattern | Distilled rule |
|---|---|
| [Canonical Protocol](https://patterns.arcitura.com/soa-patterns/design_patterns/canonical_protocol) | **One communication technology** for the inventory (e.g., HTTP+JSON, or gRPC) so services interoperate without bridging. It can become a lowest common denominator. See Dual Protocols / Augmented Protocols in `soa-inventory-and-service-implementation`. |
| [Canonical Schema](https://patterns.arcitura.com/soa-patterns/design_patterns/canonical_schema) | **Standardized data models** for the information services exchange, which avoids transformation inside the inventory. Same idea as EIP's Canonical Data Model, and it works best scoped to a domain inventory. |
| [Canonical Expression](https://patterns.arcitura.com/soa-patterns/design_patterns/canonical_expression) | Service contracts follow **naming conventions** (service, capability, and element names), so similar capabilities are expressed the same way and not misread (`getCustomer` vs `fetchClient` vs `CustomerLookup`). The conventions only help if they're enforced, so lint the API Descriptions in CI. |
| [Canonical Versioning](https://patterns.arcitura.com/soa-patterns/design_patterns/canonical_versioning) | **One versioning strategy and rules for every contract** in the inventory, so compatibility means the same thing everywhere. See `map-api-evolution-and-versioning` for the strategies. |

## Centralizing cross-cutting concerns

| Pattern | Centralize | Rule | Watch out for |
|---|---|---|---|
| [Policy Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/policy_centralization) | Global or domain policies (security, logging, QoS) | Define once and apply to many services: gateway policies, a service mesh, OPA/Rego | A single point of failure and a governance bottleneck |
| [Process Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/process_centralization) | Business process (orchestration) logic | Deploy and govern task-service process logic on a **central orchestration platform** | Platform lock-in. Process changes become platform deployments |
| [Rules Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/rules_centralization) | Business rules | Put rules in a dedicated rules service or engine (DMN, Drools), so changing a rule doesn't change several services | Performance, and the rules service becomes critical infrastructure |
| [Schema Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/schema_centralization) | Data schemas | Selected schemas exist **once** as standalone, shared definitions that many contracts reference (a schema registry) | Changing a shared schema affects every contract that uses it, so pair it with strict Canonical Versioning |
| [Metadata Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/metadata_centralization) | Service metadata | A **service registry/catalog** where services are published and discovered, so teams find a service before building a duplicate. Logic Centralization depends on it. Backstage and API portals are modern examples | A catalog that goes stale is worse than none. Generate it from the pipeline |

## Inventory design workflow

1. Choose scope: enterprise or domain inventories, and draw domain boundaries.
2. Model the inventory blueprint: candidate services by layer (utility/entity/task/micro-task),
   checked for overlap (normalization).
3. Set standards: canonical protocol, schema, naming, versioning, with automated checks.
4. Decide what to centralize (policy, rules, schemas, process), weighing reuse against
   bottleneck and blast radius.
5. Publish everything in a registry (metadata centralization) and enforce use of official services
   (logic centralization).

## Related skills

- `soa-inventory-and-service-implementation`: how individual services in the inventory are built
- `soa-service-contract-and-governance`: contract design and changing services over time
- `integration-architecture-patterns`: API-led layering, the modern view of service layers
