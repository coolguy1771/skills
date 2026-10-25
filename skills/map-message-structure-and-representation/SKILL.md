---
name: map-message-structure-and-representation
description: Design API request/response payloads using Microservice API Patterns (MAP) structure patterns — representation elements Atomic Parameter, Atomic Parameter List, Parameter Tree, Parameter Forest; element stereotypes Data Element, Metadata Element, Id Element, Link Element; and special-purpose representations API Key, Error Report, Context Representation. Use this whenever the user is designing JSON/Protobuf/GraphQL message schemas, asks "flat parameters or a nested object," "how should IDs look in my API (UUID, URN, URI)," "where do metadata and pagination info go," "should I include links (HATEOAS)," "how should API errors be structured (Problem Details/RFC 9457)," "how do I pass tenant, correlation, or auth context," or "are API keys enough for authentication." Also trigger for reviewing OpenAPI schemas and DTO design.
---

# MAP: Message Structure and Representation

After deciding what an operation does (`map-api-foundation-and-responsibilities`), decide **what
its messages look like**. These patterns give a shared vocabulary for payload shape and for the
role each element plays. That role decides how it's validated, versioned, and documented.

## Representation elements: the shape of a payload

Start simple and add structure only when the data needs it:

| Pattern | Shape | Use when | Example |
|---|---|---|---|
| [Atomic Parameter](https://microservice-api-patterns.org/patterns/structure/representationElements/AtomicParameter) | One scalar: number, string, boolean, binary | A single simple value | `GET /customers/{id}`; response `true` |
| [Atomic Parameter List](https://microservice-api-patterns.org/patterns/structure/representationElements/AtomicParameterList) | A flat group of related scalars | A few related values with no nesting. Make the grouping explicit in the contract | Query params `?from=2026-01-01&to=2026-02-01`; `{ "street":..., "zip":... }` |
| [Parameter Tree](https://microservice-api-patterns.org/patterns/structure/representationElements/ParameterTree) | A hierarchy: nested objects and arrays | **Containment** relationships (an order *contains* line items) | Typical JSON body |
| [Parameter Forest](https://microservice-api-patterns.org/patterns/structure/representationElements/ParameterForest) | Several top-level trees side by side | One message carries **several independent structures** | `{ "customer": {...}, "contracts": [...] }` |

**Rules:**
- Nest for *containment*, not for every association. Express associations to independently
  meaningful entities with Id or Link Elements (see `map-api-quality-and-governance` for embedding
  vs linking).
- Deep trees make versioning and partial responses harder. Three or four levels is a practical
  maximum before it's worth reconsidering.
- A forest is often a sign the operation does two things. Check whether it should be two operations
  or a Request Bundle.

## Element stereotypes: the role of each element

| Pattern | Role | Guidance |
|---|---|---|
| [Data Element](https://microservice-api-patterns.org/patterns/structure/elementStereotypes/DataElement) | Domain/application data | Expose a **published-language** view (API DTOs), never internal persistence entities. That decouples the provider's data management from clients. Map explicitly. |
| [Metadata Element](https://microservice-api-patterns.org/patterns/structure/elementStereotypes/MetadataElement) | Data *about* data: units, provenance, version, counts, timestamps, pagination info | Lets receivers interpret content without hard-coded assumptions. There are three kinds: **control** (flags, versions), **provenance** (origin, timestamps), and **aggregated** (counts, totals). Keep metadata clearly separate from data, e.g., a `meta` object or headers. |
| [Id Element](https://microservice-api-patterns.org/patterns/structure/elementStereotypes/IdElement) | Identifies an element at design time and at runtime | Pick the scope deliberately: local (unique per collection), global (UUID), or **human-readable URNs** (`urn:acme:order:123`). Don't leak sequential DB keys on public APIs, because it enables enumeration and exposes volume. IDs are opaque to clients, so never make them parse IDs. |
| [Link Element](https://microservice-api-patterns.org/patterns/structure/elementStereotypes/LinkElement) | A **remotely callable reference** to an endpoint or operation | A URI plus link relation (`rel`) and optionally method/media type (HAL, JSON:API, Siren, RFC 8288 `Link` header). Links let the server change URL layout and advertise *allowed state transitions* (HATEOAS). They're most valuable in public, long-lived APIs. |

## Special-purpose representations

### [API Key](https://microservice-api-patterns.org/patterns/structure/specialPurposeRepresentations/APIKey)
A provider-issued identifier that **identifies** the calling client (application) and supports
metering, rate limiting, and pricing plans.
**Rules:** an API key **identifies a client application**. It doesn't authenticate a user or
prove much on its own. Send it in a header, never in the URL (URLs end up in logs). Combine it
with a secret signature (HMAC) or use OAuth 2.0/OIDC when you need real authentication or user
delegation. Support rotation and per-key revocation.

### [Error Report](https://microservice-api-patterns.org/patterns/structure/specialPurposeRepresentations/ErrorReport)
Return a **structured, protocol-independent error payload** in addition to protocol status codes.
**Rules:** use **RFC 9457 Problem Details** (`type`, `title`, `status`, `detail`, `instance`,
plus extensions) for HTTP APIs. Include a machine-readable error code clients can branch on, a
human-readable message, field-level details for validation failures, and a correlation/trace ID.
**Never include stack traces, SQL, or internal hostnames**: that's the Exception Shielding
concern (see `soa-service-security-patterns`). Error codes are part of the contract, so version
them.

### [Context Representation](https://microservice-api-patterns.org/patterns/structure/specialPurposeRepresentations/ContextRepresentation)
Bundle **context information** (identity/tenant, correlation and conversation IDs, locale,
priority, QoS hints, request deadlines) into a defined structure in the *payload*, so it survives
protocol hops (HTTP → queue → gRPC) that would drop protocol headers.
**Rules:** decide explicitly what lives in protocol headers and what lives in the payload context.
Propagate correlation and trace IDs through *every* hop of a conversation. Validate
security-relevant context (tenant, identity) on the server, and never trust it blindly from the
client.

## Payload review checklist

1. Is each element's stereotype (data/metadata/id/link) clear and consistently placed?
2. Are internal persistence models hidden behind API-specific Data Elements?
3. Are IDs opaque, scoped deliberately, and non-enumerable on public APIs?
4. Do errors follow one structured format (Problem Details) with stable codes and no internals?
5. Does context (tenant, correlation, deadline) survive every hop?
6. Is nesting depth reasonable, and are associations linked rather than deeply embedded by default?

## Related skills

- `map-api-quality-and-governance`: Embedded Entity vs Linked Information Holder, Pagination, Wish List
- `map-api-evolution-and-versioning`: Version Identifier as a metadata element
- `eip-message-construction`: the messaging-side equivalents (Correlation Identifier, Format Indicator)
