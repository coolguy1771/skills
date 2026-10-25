---
name: soa-service-contract-and-governance
description: Design service contracts and evolve services over time using SOA patterns (Thomas Erl / Arcitura) — service contract design patterns Decoupled Contract, Contract Centralization, Concurrent Contracts, Contract Denormalization, Validation Abstraction; and service governance patterns Compatible Change, Version Identification, Termination Notification, Service Refactoring, Service Decomposition, Decomposed Capability, Distributed Capability, Proxy Capability. Use this whenever the user asks how to keep a contract independent of its implementation, why consumers must not bypass a service to hit its database, how to offer different contracts to different consumer types, whether to add coarse- and fine-grained versions of the same operation, how to stop validation rules from causing breaking changes, how to split an overgrown service without breaking consumers, how to retire a service, or how to refactor internals safely behind a stable contract.
---

# SOA: Service Contract Design and Governance

The contract is the only part of a service consumers should couple to. Contract design patterns
keep that coupling **narrow and stable**. Governance patterns let the service **change behind or
alongside the contract** without breaking anyone. Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Contract design

| Pattern | Problem | Distilled rule |
|---|---|---|
| [Decoupled Contract](https://patterns.arcitura.com/soa-patterns/design_patterns/decoupled_contract) | Contracts generated from code tie consumers to the implementation technology | Define the contract **physically separate from, and independent of, the implementation**: design-first OpenAPI/AsyncAPI/proto/XSD, then implement to it. Don't let code-first generators turn internal class shapes into the public contract. |
| [Contract Centralization](https://patterns.arcitura.com/soa-patterns/design_patterns/contract_centralization) | Consumers bypass the contract and reach the service's database or internals directly | Make the published contract the **only** entry point to the service's logic and data. Bypassing it creates hidden coupling that blocks every future change. Enforce it with network policy and per-service credentials, not documentation alone. |
| [Concurrent Contracts](https://patterns.arcitura.com/soa-patterns/design_patterns/concurrent_contracts) | One contract can't suit every consumer type (internal vs partner, different protocols or security) | Offer **several contracts for the same service logic**, each tailored to a consumer group, usually through Service Façades. Each extra contract must be governed and versioned. |
| [Contract Denormalization](https://patterns.arcitura.com/soa-patterns/design_patterns/contract_denormalization) | A strictly minimal contract forces consumers to make many calls or receive too much data | Allow **controlled redundancy**: capabilities that overlap at different granularities (e.g., `GetCustomer` and `GetCustomerSummary`) so consumers get what they need efficiently. Keep it bounded. Every redundant capability is more surface to version. |
| [Validation Abstraction](https://patterns.arcitura.com/soa-patterns/design_patterns/validation_abstraction) | Detailed validation constraints in the contract (regexes, ranges, enums) become breaking changes when business rules change | **Move volatile, granular validation out of the contract** and into the service logic (or a validation service). Keep only stable, structural constraints in the schema. The contract changes less often. The cost is that consumers learn some rules only at runtime, so return clear Error Reports. |

## Governance: changing a service over time

### Small changes

| Pattern | Distilled rule |
|---|---|
| [Compatible Change](https://patterns.arcitura.com/soa-patterns/design_patterns/compatible_change) | Make changes **backward-compatible** wherever possible: add optional elements and new capabilities, and never remove or tighten. Existing consumers keep working, with no version bump needed. |
| [Version Identification](https://patterns.arcitura.com/soa-patterns/design_patterns/version_identification) | Contracts **carry version information** that tells consumers whether a change is compatible or incompatible. Follow the inventory's Canonical Versioning rules. |
| [Service Refactoring](https://patterns.arcitura.com/soa-patterns/design_patterns/service_refactoring) | Improve the **internal logic and implementation** (performance, technology, code quality) while the **contract stays unchanged**. This is the reason for Decoupled Contract and Service Façade. Contract tests are the safety net. |
| [Termination Notification](https://patterns.arcitura.com/soa-patterns/design_patterns/termination_notification) | The contract **states when a service, capability, or contract version will be retired**, so consumers can plan. Today: `Deprecation`/`Sunset` headers, catalog lifecycle states, published policies (see `map-api-evolution-and-versioning`). |

### Splitting overgrown services

Services that grow too large (too much logic, load, or too many owners) need to be split. These
four patterns cover it, and the first is applied *at design time* to make the later split cheap:

1. **[Decomposed Capability](https://patterns.arcitura.com/soa-patterns/design_patterns/decomposed_capability)**:
   when designing a service you expect to grow (often an entity service), **group capabilities so
   they can be split out later** along clear seams. Design for decomposition before you need it.
2. **[Service Decomposition](https://patterns.arcitura.com/soa-patterns/design_patterns/service_decomposition)**:
   when the time comes, **split the service into several more fine-grained services**. That
   improves scalability and ownership but adds composition overhead (more hops, more services).
3. **[Proxy Capability](https://patterns.arcitura.com/soa-patterns/design_patterns/proxy_capability)**:
   after decomposition, **keep the original capability in the original contract as a proxy** that
   forwards to its new home, so existing consumers don't break. Mark it deprecated and migrate
   consumers. It's the contract-level version of the strangler fig.
4. **[Distributed Capability](https://patterns.arcitura.com/soa-patterns/design_patterns/distributed_capability)**:
   when one capability has **special processing needs** (heavy compute, special runtime) that
   don't fit the service's environment, **keep it in the service's contract but move its logic
   outside the service boundary**, e.g., to a dedicated component or worker. The contract stays
   intact and the service stays autonomous. The cost is an extra hop and distributed debugging.

## Contract governance checklist

1. Is the contract written design-first and versioned in source control (Decoupled Contract)?
2. Is direct database or internal access from other services blocked (Contract Centralization)?
3. Is every change classified as compatible or incompatible, and versioned by inventory rules?
4. Are volatile business rules kept out of schema constraints (Validation Abstraction)?
5. Is there a published retirement policy, and are deprecated capabilities marked in the contract?
6. Do contract tests (consumer-driven, schema diff) run in CI to catch accidental breaks?

## Related skills

- `map-api-evolution-and-versioning`: modern versioning strategies and deprecation mechanics
- `soa-inventory-and-service-implementation`: Service Façade, which enables refactoring and concurrent contracts
- `soa-service-inventory-design`: Canonical Versioning and Canonical Expression standards
