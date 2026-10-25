---
name: map-api-quality-and-governance
description: Improve API performance, efficiency, and commercial governance using Microservice API Patterns (MAP) quality patterns — reference management (Embedded Entity, Linked Information Holder), data transfer parsimony (Pagination, Wish List, Wish Template, Conditional Request, Request Bundle), and quality management (Pricing Plan, Rate Limit, Service Level Agreement). Use this whenever the user asks "should I embed related data or return links," "how do I fix chatty APIs or N+1 calls," "offset vs cursor pagination," "let clients choose which fields come back (sparse fieldsets, GraphQL-like selection)," "how do I avoid re-sending unchanged data (ETags, 304)," "batch requests," "how do I rate-limit or meter my API," "how should we price API usage," or "what should our API SLA/SLO say." Also trigger for API performance reviews and API product management.
---

# MAP: API Quality and Governance

Once an API works, these patterns make it **efficient** (fewer and smaller messages) and
**governable** (metered, protected, with clear promises). Every pattern here trades message size
against message count, or provider control against client freedom. Choose based on real client
usage, not in the abstract.

## Reference management: embed or link?

| Pattern | Do this | Good when | Costs |
|---|---|---|---|
| [Embedded Entity](https://microservice-api-patterns.org/patterns/quality/referenceManagement/EmbeddedEntity) | Put the related data **inside** the response | Clients almost always need the related data together; it's small; it changes at the same rate | Larger messages, data some clients don't need, and duplication if the same entity appears many times |
| [Linked Information Holder](https://microservice-api-patterns.org/patterns/quality/referenceManagement/LinkedInformationHolder) | Return a **Link Element** to a separate endpoint that serves the related data | Related data is large, rarely needed, changes independently, or is shared widely | More round trips (the N+1 risk) and the need to resolve links |

**Rule:** default to linking for independently owned aggregates and to embedding for owned
children (value objects, line items). If clients disagree, let them choose with a Wish List
(`?expand=customer`).

## Data transfer parsimony

| Pattern | Problem | Distilled rule |
|---|---|---|
| [Pagination](https://microservice-api-patterns.org/patterns/quality/dataTransferParsimony/Pagination) | Large result sets overwhelm clients and servers | Return data in pages. **Cursor-based** pagination is stable under concurrent inserts and efficient on large data, so it's the default for feeds and big collections. **Offset-based** pagination allows random access (`page=7`) but drifts and slows at deep offsets. Return page metadata (next cursor/link, maybe a total count, which can be expensive) and enforce a max page size on the server. |
| [Wish List](https://microservice-api-patterns.org/patterns/quality/dataTransferParsimony/WishList) | Clients need different subsets of fields | The client lists the **fields or related entities it wants** (`?fields=id,name`, `?expand=items`). Simple, cacheable per variant, and covers most needs. Validate requested names against an allow-list. |
| [Wish Template](https://microservice-api-patterns.org/patterns/quality/dataTransferParsimony/WishTemplate) | Clients need to choose **nested** data flexibly | The client sends a **mock/template of the response structure** marking what it wants. GraphQL selection sets are the full form of this. More powerful than a Wish List, but harder to cache and secure, and clients can request expensive queries, so add query cost limits. |
| [Conditional Request](https://microservice-api-patterns.org/patterns/quality/dataTransferParsimony/ConditionalRequest) | Clients repeatedly fetch data that rarely changes | The client sends a fingerprint or timestamp (`If-None-Match: <ETag>`, `If-Modified-Since`). The server returns `304 Not Modified` without a body if nothing changed. It saves bandwidth, and server work if the ETag is cheap to compute. On writes, `If-Match` doubles as optimistic concurrency control. |
| [Request Bundle](https://microservice-api-patterns.org/patterns/quality/dataTransferParsimony/RequestBundle) | Many small requests waste round trips | Combine several requests in one message and return the responses together (batch endpoints, JSON:API atomic ops). Define the **error semantics**: all-or-nothing vs per-item status (e.g., `207 Multi-Status`). Cap the bundle size. Bundles complicate caching and rate limiting. |

**Which to reach for:**
- Too many calls → Embedded Entity, Request Bundle, Wish List with expand.
- Too much data per call → Pagination, Wish List/Template, Linked Information Holder.
- The same data fetched repeatedly → Conditional Request plus HTTP caching.

## Quality management and governance

### [Rate Limit](https://microservice-api-patterns.org/patterns/quality/qualityManagementAndGovernance/RateLimit)
Cap how much each client can call (requests or data volume per time window) to protect the
provider and give clients fair shares.
**Rules:** limit **per client identity** (API key or token subject), not per IP alone. Use token
bucket or sliding window. Return `429 Too Many Requests` with `Retry-After` and expose
remaining quota (`RateLimit-*` headers), so clients can back off gracefully. Set limits from
measured capacity and tie them to pricing tiers. Document them in the API Description, because
clients design around them.

### [Pricing Plan](https://microservice-api-patterns.org/patterns/quality/qualityManagementAndGovernance/PricingPlan)
Meter usage and charge for it: **flat-rate subscription**, **usage-based (pay per call/unit)**,
**freemium tiers**, or **market-based** pricing.
**Rules:** metering must be accurate, auditable, and tamper-resistant. Base it on authenticated
identities (API Key). Make tiers correspond to rate limits and SLAs. Give clients a usage
dashboard, because unexpected bills destroy trust faster than outages.

### [Service Level Agreement](https://microservice-api-patterns.org/patterns/quality/qualityManagementAndGovernance/ServiceLevelAgreement)
State the API's quality characteristics in **measurable terms**, along with what happens when
they're missed.
**Rules:** each objective needs a metric, a measurement method, a time window, and a target, e.g.,
"99.9% of requests per calendar month return non-5xx" or "p95 latency < 300 ms measured at the
gateway." Internally, run with tighter SLOs and error budgets than the external SLA promises. Say
what's excluded (maintenance windows, client-caused errors) and what the remedy is (credits).
Public and community APIs need written SLAs. Internal APIs benefit from SLOs even without a
contract.

## Related skills

- `map-message-structure-and-representation`: Link Element, API Key, and Metadata Element, which these patterns build on
- `setup-rate-limiting`: implementing rate limiting in code and gateways
- `implement-caching-strategy`: server- and edge-side caching that complements Conditional Request
