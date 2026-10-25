---
name: map-api-foundation-and-responsibilities
description: Scope and shape APIs using Microservice API Patterns (MAP) foundation and responsibility patterns — Frontend Integration, Backend Integration, Public API, Community API, Solution-Internal API, API Description; endpoint roles Processing Resource and Information Holder Resource; operation responsibilities State Creation Operation, Retrieval Operation, State Transition Operation, Computation Function; and information holder types Operational Data Holder, Master Data Holder, Reference Data Holder, Link Lookup Resource, Data Transfer Resource. Use this whenever the user is designing a new API or endpoint set, asks "who is this API for and how open should it be," "is this endpoint an action or a resource," "should this be a POST, PUT, or PATCH," "how do I model a business process as an API," "how should master data or reference data be exposed," or needs to write an OpenAPI/AsyncAPI description. Also trigger for API design reviews and decomposing a domain model into API endpoints.
---

# MAP: API Foundations and Endpoint Responsibilities

[Microservice API Patterns](https://microservice-api-patterns.org) (Zimmermann, Stocker, Lübke,
Zdun, Pautasso; *Patterns for API Design*) give names to API design decisions. This skill covers
the first two questions in any API design: **what kind of API is this?** and **what is each
endpoint and operation responsible for?**

## 1. Foundation: direction and visibility

**Direction: who calls whom?**

| Pattern | Distilled rule |
|---|---|
| [Frontend Integration](https://microservice-api-patterns.org/patterns/foundation/FrontendIntegration) | The API serves **UI clients** (web, mobile) that are physically separate from back-end logic. Optimize for the screens: fewer round trips, payloads shaped for views, and tolerance for flaky networks. Often a BFF per client type. |
| [Backend Integration](https://microservice-api-patterns.org/patterns/foundation/BackendIntegration) | The API connects **independently built and deployed back-end systems** that exchange data and trigger each other. Optimize for **loose coupling and each side's conceptual integrity**: no shared internal models, explicit contracts, and asynchronous messaging where possible. |

**Visibility: how open is it?**

| Pattern | Audience | Governance implications |
|---|---|---|
| [Public API](https://microservice-api-patterns.org/patterns/foundation/PublicAPI) | Unlimited/unknown external clients worldwide | Strongest compatibility promises, API keys/OAuth, rate limits, pricing plans, excellent docs. You can't coordinate with clients, so breaking changes are very expensive |
| [Community API](https://microservice-api-patterns.org/patterns/foundation/CommunityAPI) | A **closed group of several legal entities** (partners, consortium, gov agencies) | Access restricted by contract and identity federation. You can negotiate changes, but slowly. SLAs often contractual |
| [Solution-Internal API](https://microservice-api-patterns.org/patterns/foundation/SolutionInternalAPI) | Components **within one application/solution** | Can change in lockstep with its consumers. Lighter docs and versioning, but still a contract once teams split |

Visibility decides how much of the quality and evolution patterns you need. Public APIs need most
of `map-api-quality-and-governance` and `map-api-evolution-and-versioning`. Solution-internal APIs
can skip much of it.

**[API Description](https://microservice-api-patterns.org/patterns/foundation/APIDescription)**:
share a **functional contract** (endpoints, operations, message syntax: OpenAPI, AsyncAPI, gRPC
proto, GraphQL SDL) plus **semantics and quality** (what operations mean, pre/postconditions,
error behaviour, SLAs, rate limits, lifecycle). Minimal descriptions (syntax only) get you
started. Elaborate ones (with semantics and examples) are what make external consumers
successful. Treat the description as source-controlled and versioned, ideally design-first.

## 2. Endpoint roles

| Pattern | The endpoint represents | Example |
|---|---|---|
| [Processing Resource](https://microservice-api-patterns.org/patterns/responsibility/endpointRoles/ProcessingResource) | An **activity**. Clients ask the provider to *do something* | `POST /claims/{id}/assessments`, `POST /quotes:calculate` |
| [Information Holder Resource](https://microservice-api-patterns.org/patterns/responsibility/endpointRoles/InformationHolderResource) | **Domain data**. Clients read and modify entities through it, with the implementation hidden | `GET/PUT /customers/{id}` |

Information holders must protect **data integrity under concurrent access**, using optimistic
locking (ETags/`If-Match`), validation, and invariants that the server enforces. Don't expose
your database table structure 1:1. That's the "CRUD-over-tables" anti-pattern, and it couples
clients to your schema.

## 3. Operation responsibilities

Classify each operation by **whether it reads or writes provider-side state**:

| Pattern | Reads state? | Writes state? | Typical HTTP | Notes |
|---|---|---|---|---|
| [State Creation Operation](https://microservice-api-patterns.org/patterns/responsibility/operationResponsibilities/StateCreationOperation) | No (write-only) | Yes | `POST` | A client reports that something happened ("event received"). Often accepted for later processing (`202`). Make it idempotent with client-supplied IDs or idempotency keys |
| [Retrieval Operation](https://microservice-api-patterns.org/patterns/responsibility/operationResponsibilities/RetrievalOperation) | Yes | No | `GET` | Read-only queries and searches. Combine with Pagination, Wish List, Conditional Request |
| [State Transition Operation](https://microservice-api-patterns.org/patterns/responsibility/operationResponsibilities/StateTransitionOperation) | Yes | Yes | `PUT`/`PATCH`/`POST` action/`DELETE` | Moves a business object or process through its lifecycle (`submit`, `approve`, `cancel`). Model the **state machine** explicitly, and reject illegal transitions with a clear Error Report. Full replace (`PUT`) vs partial update (`PATCH`) vs business action (`POST /orders/{id}/cancellation`) is a deliberate choice |
| [Computation Function](https://microservice-api-patterns.org/patterns/responsibility/operationResponsibilities/ComputationFunction) | No | No | `GET` or `POST` | A **side-effect-free** calculation from input to output (validation, pricing, conversion). Safe to cache and retry. Ask whether it belongs in a client library instead |

**Rule of thumb:** prefer **business-activity-oriented transitions** (`approve`) over generic
field updates (`PATCH status=approved`). The activity version keeps business rules on the server
and makes the lifecycle visible in the API.

## 4. Information holder types

Choose by the data's **lifetime, mutability, and number of references**:

| Pattern | Data character | Design guidance |
|---|---|---|
| [Operational Data Holder](https://microservice-api-patterns.org/patterns/responsibility/informationHolderEndpointTypes/OperationalDataHolder) | Short-lived, changes often, many outgoing relations (orders, shipments) | Full CRUD plus state transitions. Expect high write volume and concurrency. Link to master data rather than embedding it |
| [Master Data Holder](https://microservice-api-patterns.org/patterns/responsibility/informationHolderEndpointTypes/MasterDataHolder) | Long-lived, changes rarely, referenced by many (customers, products) | Strong integrity and audit. Often owned by an MDM system. Cache-friendly reads, and tightly controlled writes |
| [Reference Data Holder](https://microservice-api-patterns.org/patterns/responsibility/informationHolderEndpointTypes/ReferenceDataHolder) | Long-lived, **immutable for clients** (country codes, currencies, status codes) | Read-only. Cache aggressively (long TTL, ETags). Clients may replicate it locally. Consider a single central reference service |
| [Link Lookup Resource](https://microservice-api-patterns.org/patterns/responsibility/informationHolderEndpointTypes/LinkLookupResource) | **Addresses** of other endpoints that may change | A directory clients query to resolve logical names to current endpoint addresses, so payloads aren't bound to physical URLs. It's an extra call and a single point of failure, so cache the results |
| [Data Transfer Resource](https://microservice-api-patterns.org/patterns/responsibility/informationHolderEndpointTypes/DataTransferResource) | Data exchanged between parties that **don't know each other or aren't online at the same time** | A shared resource that some participants `PUT/POST` to and others `GET` from later. Messaging semantics over HTTP. Define ownership, access control, and cleanup (TTL) |

## Design workflow

1. Classify the API: direction (frontend/backend) × visibility (public/community/internal).
2. Find candidate endpoints from the domain model (aggregates become information holders,
   activities become processing resources).
3. Assign each operation a responsibility, which gives you its HTTP method, idempotency, and
   cacheability.
4. Type each information holder (operational/master/reference), which gives you its caching and
   write policy.
5. Write the API Description, then apply structure (`map-message-structure-and-representation`),
   quality (`map-api-quality-and-governance`), and evolution (`map-api-evolution-and-versioning`)
   patterns.

## Related skills

- `design-rest-api`: concrete REST conventions once the responsibilities are clear
- `integration-architecture-patterns`: API-led layering across the enterprise
