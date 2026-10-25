---
name: map-api-evolution-and-versioning
description: Evolve APIs without breaking clients using Microservice API Patterns (MAP) evolution patterns — Version Identifier, Semantic Versioning, Two In Production, Aggressive Obsolescence, Experimental Preview, Limited Lifetime Guarantee, and Eternal Lifetime Guarantee. Use this whenever the user asks "how should I version my API (URL, header, media type)," "is this change breaking," "how long must we support v1," "how do we deprecate and sunset an endpoint," "how do we release a beta API without freezing it," "how many versions should run in production," or is writing an API lifecycle/deprecation policy. Also trigger for Sunset/Deprecation headers, schema compatibility rules, and API changelogs.
---

# MAP: API Evolution and Versioning

An API is a promise. Evolution patterns decide **how changes are signaled** and **how long old
promises are kept**. Choose them deliberately, and publish them, before v1 ships. Retrofitting a
lifecycle policy after clients depend on you is far harder.

## Signaling change

### [Version Identifier](https://microservice-api-patterns.org/patterns/evolution/VersionIdentifier)
Put an explicit version in the API so clients, and the provider, can detect incompatible changes
instead of misreading data silently.

| Placement | Example | Trade-off |
|---|---|---|
| URL path | `/v2/orders` | Most visible and easiest to route and cache. Versions the *whole* API, so URLs change |
| Header / media type | `Accept: application/vnd.acme.order.v2+json` | Clean URLs and per-resource versions. Harder to test in a browser and easier to forget |
| Query param | `?api-version=2026-10-01` | Easy to use, and common for date-based versions (Azure style) |
| In the message | `"schemaVersion": 2` | Required for async messages and payloads that are stored |

**Rule:** only bump the version for **breaking** changes. Make non-breaking changes in place,
following the **tolerant reader** principle: clients ignore unknown fields, and providers only
*add* optional fields.

### [Semantic Versioning](https://microservice-api-patterns.org/patterns/evolution/SemanticVersioning)
Use `MAJOR.MINOR.PATCH`: **major** = breaking, **minor** = backward-compatible additions,
**patch** = compatible fixes. Clients can tell compatibility at a glance. Usually only the
**major** version is exposed in the URL or Version Identifier. Minor and patch belong in docs,
changelogs, and the API Description.

**What counts as breaking** (needs a major bump): removing or renaming fields, operations, or enum
values clients send; making optional input required; changing types, formats, or semantics;
tightening validation; changing error codes clients branch on. *Usually* not breaking: adding
optional fields or new endpoints. Adding enum values in *responses* is not breaking only if
clients were told to tolerate unknown values.

## Running versions side by side

### [Two In Production](https://microservice-api-patterns.org/patterns/evolution/TwoInProduction)
Run **exactly two major versions** at once: current and previous. Releasing N+1 starts the
retirement of N−1. Clients get a predictable migration window, and the provider avoids
maintaining an unbounded number of versions. It's the pragmatic default for most organizations.
The gateway routes by Version Identifier.

## Lifetime promises: pick one per API and publish it

| Pattern | Promise | Fits | Cost |
|---|---|---|---|
| [Experimental Preview](https://microservice-api-patterns.org/patterns/evolution/ExperimentalPreview) | **None**. May change or vanish at any time. For early adopter feedback | New APIs whose design isn't settled | Needs clear labeling (`/preview/`, `-beta` versions, opt-in headers) and explicit opt-in, so nobody builds production systems on it by accident |
| [Aggressive Obsolescence](https://microservice-api-patterns.org/patterns/evolution/AggressiveObsolescence) | Features are **deprecated and removed on a short, announced schedule** | Providers that must cut maintenance cost, and fast-moving domains | Clients must keep migrating. Communicate early (Deprecation/Sunset headers, changelogs, emails), and track usage of deprecated features before removing them |
| [Limited Lifetime Guarantee](https://microservice-api-patterns.org/patterns/evolution/LimitedLifetimeGuarantee) | Each published version is supported **for a fixed, stated period** (e.g., 24 months from release) | Public/community APIs with enterprise clients who need planning certainty | Provider runs several versions until each expires. Pair with Two In Production to cap how many |
| [Eternal Lifetime Guarantee](https://microservice-api-patterns.org/patterns/evolution/EternalLifetimeGuarantee) | A version is **never retired** | Clients that can't migrate (embedded devices, regulated integrations, published data formats) | Maintenance cost grows forever. Use it rarely, and design that API extremely carefully, because mistakes become permanent |

## Deprecation playbook

1. Announce in advance with a date: `Deprecation` header (RFC 9745) on responses, `Sunset` header
   (RFC 8594) with the removal date, a changelog entry, and direct notice to registered clients.
2. Measure: log deprecated-feature usage per API key, and contact remaining callers directly.
3. Optionally brown out (scheduled short outages of the old version) before final removal, to
   flush out forgotten callers.
4. Remove, then return `410 Gone` with an Error Report pointing to the migration guide, rather
   than a bare 404.

## Choosing a strategy

- Internal API with known consumers: in-place compatible changes + Two In Production for the rare
  major version.
- Public API: Semantic Versioning + Limited Lifetime Guarantee + Two In Production, and
  Experimental Preview for anything new.
- Long-lived device and partner integrations: Eternal Lifetime Guarantee for a very small, very
  stable core, and versioned extensions around it.

## Related skills

- `soa-service-contract-and-governance`: Compatible Change, Concurrent Contracts, Version
  Identification, and Termination Notification, the SOA-era equivalents
- `eip-message-construction`: Format Indicator for versioning async messages
- `map-api-quality-and-governance`: SLAs that lifetime guarantees feed into
