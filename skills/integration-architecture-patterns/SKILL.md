---
name: integration-architecture-patterns
description: Apply vendor-neutral integration architecture patterns at enterprise-architect altitude — API-led Connectivity (system/process/experience API layers), Anti-Corruption Layer, Change Data Capture, Hybrid API Management (central control plane, distributed gateways), and Hybrid Integration (on-prem + cloud integration). Use this whenever the user is designing how many systems connect across an enterprise, asks "how should we layer our APIs," "how do we integrate with a legacy system without its model leaking into ours," "how do we stream database changes to other systems without dual writes," "how do we manage APIs across cloud and on-prem gateways," or "how do we integrate SaaS, cloud, and on-prem apps." Also trigger for MuleSoft/Apigee/Kong/APIM/WSO2 topology design and strangler-fig modernization.
---

# Integration Architecture Patterns

EIP patterns describe individual message flows. These patterns describe the **shape of the whole
integration landscape**: how APIs are layered, where model boundaries sit, how data changes
spread, and how hybrid estates are governed. Source: Chanaka Fernando's vendor-neutral
[solution-architecture-patterns](https://github.com/chanakaudaya/solution-architecture-patterns).

## [API-led Connectivity](https://github.com/chanakaudaya/solution-architecture-patterns/blob/master/vendor-neutral/API-led-Connectivity-Pattern.md)

**Problem:** point-to-point integrations between every system create a brittle, invisible web.
Every new consumer re-integrates with the same back ends.
**Solution:** connect systems through **three layers of reusable APIs**:

| Layer | Purpose | Owned by | Changes when |
|---|---|---|---|
| **System APIs** | Expose systems of record (ERP, CRM, DBs, mainframe) behind a stable interface that hides their quirks | Teams that own those systems | The back end changes |
| **Process APIs** | Combine system APIs into business capabilities (e.g., "order fulfilment") | Domain/integration teams | Business processes change |
| **Experience APIs** | Tailor data for a specific channel (mobile, web, partner) — BFF style | Channel teams | UX/channel needs change |

**Rules:** each layer calls only the layer below it. Never let an experience API call a system of
record directly. Reuse is the payoff, so publish system APIs in a catalog. **Pitfall:** applying
all three layers to a simple integration adds latency and extra deployments for no gain. Collapse
layers when there's only one consumer and no foreseeable reuse.

## [Anti-Corruption Layer](https://github.com/chanakaudaya/solution-architecture-patterns/blob/master/vendor-neutral/Anti-Corruption-Layer-Pattern.md)

**Problem:** integrating with a legacy or external system whose model (names, semantics,
data shapes) differs from yours. Without a boundary, its concepts leak into your domain and spread.
**Solution:** put a **translation layer** between the two. It converts requests and responses in
both directions, so your domain only ever sees its own model. In practice it's a facade plus
adapters plus translators.
**Use when:** modernizing with the strangler fig pattern, integrating a third-party SaaS whose
model you don't control, or putting a boundary between bounded contexts with different languages.
**Rules:** the ACL belongs to the *consumer's* side, protecting your model. Keep it free of
business logic. It translates, and that's all. Plan to retire it once the legacy system is gone,
or it becomes permanent legacy itself. It adds a hop and must scale with the traffic it fronts.

## [Change Data Capture](https://github.com/chanakaudaya/solution-architecture-patterns/blob/master/vendor-neutral/Introduction-to-Change-Data-Capture.md)

**Problem:** other systems need to react to changes in a database in near real time. **Dual
writes** (app writes the DB *and* publishes an event) go out of sync whenever one write fails.
**Solution:** capture row-level changes from the database itself and publish them as a change
stream.

| CDC approach | How | Trade-off |
|---|---|---|
| **Log-based** (preferred) | Read the transaction log: MySQL binlog, Postgres WAL, SQL Server CDC, Oracle redo (Debezium, DMS, Fivetran) | Low impact on the source, captures deletes, keeps commit order. Needs log access and retention |
| **Trigger-based** | DB triggers write to shadow tables | Works anywhere, but adds write overhead to the source |
| **Query/timestamp-based** | Poll for `updated_at > last_seen` | Simple. Misses deletes and intermediate states, and adds polling load |

**Rules:** raw CDC events expose your **physical schema**, which turns your table layout into a
public contract. For cross-team consumers, prefer the **transactional outbox + CDC on the outbox
table**, so the published event is a deliberate domain event. Plan how to bootstrap with an
initial snapshot, how to handle schema changes, and how to keep ordering per key.

## [Hybrid API Management](https://github.com/chanakaudaya/solution-architecture-patterns/blob/master/vendor-neutral/Hybrid-API-Management-Pattern.md)

**Problem:** APIs are deployed across several clouds and on-prem data centers. Managing each
gateway separately fragments security policy, the catalog, and analytics.
**Solution:** separate the **control plane** (API design, publishing, policy, developer portal,
analytics) from the **data plane** (gateways). Run one central control plane and deploy gateways
**close to the back ends** they front, in each environment.
**Rules:** traffic shouldn't hairpin through a central region. Gateways must keep enforcing
cached policy if the control plane is unreachable. Sync configuration through GitOps or
APIOps pipelines, not hand edits. Examples: Azure APIM self-hosted gateways, Apigee hybrid, Kong
Konnect, WSO2 with distributed gateways.

## [Hybrid Integration](https://github.com/chanakaudaya/solution-architecture-patterns/blob/master/vendor-neutral/Hybrid-Integration-Pattern.md)

**Problem:** the estate mixes on-prem systems, cloud services, and SaaS. No single integration
technology fits every connection, and moving to the cloud is gradual.
**Solution:** combine integration styles deliberately: APIs for synchronous request/response,
messaging/event streaming for asynchronous decoupling, ETL/ELT or CDC for bulk data, and
B2B/managed file transfer for partners. Connect on-prem to cloud with secure connectivity
(private links, VPN, relay/agent-based connectors, so you avoid inbound firewall holes).
**Rules:** standardize on a **small toolkit with clear rules for which style to use when**, not
every tool for every team. Govern centrally (catalog, security, observability) and run
integrations locally. Lean toward cloud-hosted integration with on-prem agents, so connectors move
with the workloads as they migrate.

## Choosing among them

| Situation | Start with |
|---|---|
| Many consumers re-integrating the same back ends | API-led Connectivity |
| A foreign/legacy model threatening your domain | Anti-Corruption Layer |
| Downstream systems need DB changes, and you're tempted by dual writes | CDC (+ outbox) |
| APIs spread across clouds and DCs with inconsistent governance | Hybrid API Management |
| Mixed SaaS/cloud/on-prem estate with no integration strategy | Hybrid Integration |

## Related skills

- `eip-message-transformation`: Canonical Data Model and translators inside an ACL
- `map-api-foundation-and-responsibilities`: endpoint design within each API layer
- `soa-legacy-encapsulation-and-transformation`: Legacy Wrapper and Protocol Bridging
- `integration-patterns-selection-guide`: overall map of pattern families
