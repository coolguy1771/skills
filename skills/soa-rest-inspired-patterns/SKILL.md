---
name: soa-rest-inspired-patterns
description: Apply REST-inspired SOA patterns (Thomas Erl / Arcitura) — Reusable Contract (uniform contract), Entity Linking, Lightweight Endpoint, Content Negotiation, Endpoint Redirection, and Idempotent Capability. Use this whenever the user is designing RESTful services in a service inventory, asks "why use a uniform interface (GET/PUT/POST/DELETE) instead of custom operations," "how do I link related resources," "should I expose fine-grained resources for simple reads," "how do I serve JSON and XML (or v1/v2 representations) from one endpoint," "how do I move a service without breaking consumers (301/308 redirects)," or "how do I make retries safe (idempotency keys, PUT vs POST)." Also trigger for HATEOAS and hypermedia discussions and REST API reviews.
---

# SOA: REST-Inspired Patterns

These patterns come from applying REST constraints to service inventory design. Their common theme
is **moving coupling away from service-specific contracts** and onto **uniform, generic mechanisms**
(standard methods, media types, links, status codes), so consumers depend on less that can change.
Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## [Reusable Contract](https://patterns.arcitura.com/soa-patterns/design_patterns/reusable_contract)

**Problem:** with service-specific contracts, every consumer couples to every contract it uses.
Changes break it, and composing a new service means new contract-specific work.
**Solution:** limit tight coupling to **one common, reusable technical contract shared by many
services**, with generic, high-level methods that rarely change, like HTTP's `GET`/`PUT`/`POST`/`DELETE`
applied to resource identifiers.
**Application:** methods are organized around *types of information exchange*, not business
functions. Business context comes from **which resource** the consumer targets and **which media
type** it exchanges.
**Impacts:** getting the uniform contract right becomes critical, because every service depends on
it. Governance effort rises, and generic methods carry little discovery metadata, so maintain
service-specific metadata separately (API Descriptions, a catalog).

## [Entity Linking](https://patterns.arcitura.com/soa-patterns/design_patterns/entity_linking)

Entities are related, but consumers otherwise need hard-coded knowledge of how to navigate between
the services that own them. **Entity services include links to related entities' resources** in
their responses (`order` → `customer` link), so consumers follow links instead of building URLs.
This is the service-inventory version of HATEOAS. It works with the Link Element and Linked
Information Holder patterns in `map-message-structure-and-representation` and
`map-api-quality-and-governance`. Use stable link relations (`rel`) as the contract, not URL
patterns.

## [Lightweight Endpoint](https://patterns.arcitura.com/soa-patterns/design_patterns/lightweight_endpoint)

Coarse-grained capabilities make consumers pay for processing and data they don't need when they
only want a simple piece of information. Provide **lightweight, fine-grained endpoints** that give
direct, cheap access to specific resources or data, alongside the richer capabilities. Good for
high-volume simple reads. Watch for endpoint sprawl, and don't let lightweight endpoints bypass
business rules on writes.

## [Content Negotiation](https://patterns.arcitura.com/soa-patterns/design_patterns/content_negotiation)

Consumers need the same resource in different formats or representations. **One endpoint serves
several representations**, chosen per request through negotiation (`Accept`,
`Accept-Language`, `Accept-Encoding` → `Content-Type`). Use it for format (JSON/XML/CSV),
representation versions (vendor media types such as `application/vnd.acme.v2+json`), and language.
Return `406 Not Acceptable` for unsupported requests, set `Vary` headers so caches stay correct,
and document which media types are supported.

## [Endpoint Redirection](https://patterns.arcitura.com/soa-patterns/design_patterns/endpoint_redirection)

Services move (URL restructuring, decomposition, migration), which breaks consumers holding old
addresses. **The old endpoint redirects consumers** to the new location with standard status codes:
`301`/`308` for permanent moves (`308` preserves the method and body), and `302`/`307` for
temporary ones. Consumers must follow redirects, and well-behaved ones update stored addresses after
a permanent redirect. Pair it with Termination Notification for the old address's eventual removal,
and with Proxy Capability when a redirect isn't possible (e.g., non-HTTP consumers).

## [Idempotent Capability](https://patterns.arcitura.com/soa-patterns/design_patterns/idempotent_capability)

Networks fail mid-request, so consumers can't tell whether a request was processed and must
retry. Design capabilities so **repeating the same request has the same effect as doing it once**,
which makes retries safe.
- `GET`, `PUT`, and `DELETE` are idempotent by HTTP semantics. Make sure your implementation honours that.
- For `POST` (creation, payments), accept an **idempotency key** from the client, store the result
  keyed by it, and return the stored result on a retry.
- Prefer *set-to-value* operations over *increment* operations.
- Conditional requests (`If-Match`) prevent lost updates when retries interleave with other writers.

## How the patterns fit together

A REST-inspired inventory typically uses a **Reusable Contract** (HTTP's uniform interface),
**Entity Linking** for navigation, **Content Negotiation** for representations and versions,
**Idempotent Capabilities** for safe retries, **Endpoint Redirection** for moves, and **Lightweight
Endpoints** for cheap reads. Together they push consumer coupling down to standardized HTTP
semantics and link relations.

## Related skills

- `map-api-foundation-and-responsibilities`: operation responsibilities, which map onto HTTP methods
- `map-api-evolution-and-versioning`: versioning through content negotiation and deprecation
- `design-rest-api`: practical REST API design conventions
