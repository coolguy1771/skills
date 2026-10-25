---
name: integration-patterns-selection-guide
description: Entry point for choosing integration, messaging, API, and SOA design patterns — maps a concrete integration problem to the right pattern family (Enterprise Integration Patterns, integration architecture patterns, Microservice API Patterns, SOA patterns) and the specific skill to load next. Use this whenever the user describes an integration or distributed-system design problem without naming a pattern ("how should these systems talk," "we keep getting duplicate messages," "our API is chatty," "how do we version this," "how do we wrap this mainframe," "saga or distributed transaction"), asks "which pattern should I use for X," wants a pattern vocabulary for an architecture review, or is comparing EIP vs MAP vs SOA approaches.
---

# Integration Patterns Selection Guide

There are four pattern catalogs, each at a different altitude. Find the problem below, then load
the linked skill for the distilled patterns.

| Catalog | Altitude | Answers |
|---|---|---|
| **Enterprise Integration Patterns** (Hohpe & Woolf) | Individual message flows | How messages are built, carried, routed, transformed, consumed, and observed |
| **Integration Architecture Patterns** | Enterprise landscape | How APIs are layered, where model boundaries sit, how changes spread across hybrid estates |
| **Microservice API Patterns** (Zimmermann et al.) | One API's contract | What endpoints and operations do, payload shape, efficiency, evolution |
| **SOA Patterns** (Erl / Arcitura) | Service portfolio and its services | Inventory scope and layers, service implementation, contracts, security, composition |

## Problem → skill

### "How should these systems communicate?"
- Queue vs topic, channel design, dead letters, delivery guarantees → `eip-messaging-channels-and-infrastructure`
- Command vs event vs document, request-reply, correlation, expiry, schema version field → `eip-message-construction`
- Sync call vs queue vs event vs callback at the architecture level → `soa-service-messaging-patterns`
- Mixed SaaS/cloud/on-prem estate → `integration-architecture-patterns` (Hybrid Integration)

### "Messages need to go to the right place / be combined"
- Route by content, broadcast, split/aggregate, reorder, multi-step workflows → `eip-message-routing`

### "Their data doesn't match ours"
- Enrich, filter, wrap, large payloads (Claim Check), many input formats, canonical model → `eip-message-transformation`
- Model/format/protocol conversion between services, legacy wrappers, flat files → `soa-legacy-encapsulation-and-transformation`
- Keep a foreign model out of our domain → `integration-architecture-patterns` (Anti-Corruption Layer)

### "Our consumers are fragile"
- Duplicates, at-least-once, outbox, scaling consumers, ordering, push vs pull → `eip-messaging-endpoints-and-consumers`

### "We can't see what's happening"
- Tapping, tracing, auditing, synthetic messages, purging, control channels → `eip-system-management`

### "We're designing an API"
- Audience/visibility, endpoint and operation roles, master vs reference data → `map-api-foundation-and-responsibilities`
- Payload shape, IDs, links, error format, API keys, context propagation → `map-message-structure-and-representation`
- Embed vs link, pagination, field selection, ETags, batching, rate limits, pricing, SLAs → `map-api-quality-and-governance`
- Versioning, breaking changes, deprecation, lifetime guarantees → `map-api-evolution-and-versioning`
- Uniform contract, hypermedia links, content negotiation, redirects, idempotent retries → `soa-rest-inspired-patterns`

### "We're designing many services"
- Inventory scope, layers (entity/task/utility/micro), canonical standards, centralizing rules/policies/schemas, registry → `soa-service-inventory-design`
- What becomes a service, façades, state and scale, replicas, external endpoints, containers → `soa-inventory-and-service-implementation`
- Contract independence, multiple contracts, splitting services, retiring services → `soa-service-contract-and-governance`
- Change data across systems without dual writes, API layering, hybrid gateways → `integration-architecture-patterns`

### "Multi-service operations"
- Orchestration, sagas vs atomic transactions, reusable sub-orchestrations, composition availability → `soa-service-composition-patterns`
- Messaging-level Process Manager and Routing Slip → `eip-message-routing`

### "Is it secure?"
- Error leakage, payload screening, perimeter, service identity, authentication, encryption, signing → `soa-service-security-patterns`

## The same idea in different catalogs

The catalogs overlap. When two sources describe the same idea, use the vocabulary your audience
knows:

| Concept | EIP | MAP | SOA / other |
|---|---|---|---|
| Shared data model | Canonical Data Model | Data Element (published language) | Canonical Schema |
| Schema/version marker | Format Indicator | Version Identifier | Version Identification, Canonical Versioning |
| Safe duplicates/retries | Idempotent Receiver | (State Creation idempotency keys) | Idempotent Capability |
| Large payload by reference | Claim Check | Linked Information Holder | (Data Transfer Resource) |
| Central orchestration | Process Manager | Processing Resource + State Transition | Process Abstraction, Process Centralization |
| Wrapping a non-native system | Channel Adapter, Messaging Gateway | — | Legacy Wrapper, Service Façade |
| Translating models | Message Translator, Normalizer | — | Data Model Transformation, Anti-Corruption Layer |
| Async reply addressing | Return Address + Correlation Identifier | Context Representation | Service Callback |
| Message metadata | Message header, Envelope Wrapper | Metadata Element | Messaging Metadata |
| Event notification | Event Message + Publish-Subscribe Channel | — | Event-Driven Messaging |

## Common anti-patterns these patterns prevent

- **Distributed monolith**: synchronous chains plus a shared database. See Contract Centralization, Service Data Replication, Asynchronous Queuing.
- **Smart pipes**: business logic in routers or ESB/gateway policies. Keep routers to *where*, not *what*.
- **Dual writes**: write the DB and publish separately. Use the transactional outbox + CDC.
- **Unowned DLQs**: dead-letter channels nobody monitors.
- **CRUD-over-tables APIs**: exposing persistence models directly. Use Data Elements and State Transition Operations.
- **Version-free schemas**: no Format Indicator or Version Identifier until the first breaking change.
