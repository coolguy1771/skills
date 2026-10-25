---
name: soa-legacy-encapsulation-and-transformation
description: Bring legacy systems and mismatched services into a service architecture using SOA patterns (Thomas Erl / Arcitura) — legacy encapsulation patterns Legacy Wrapper, File Gateway, Multi-Channel Endpoint; and transformation patterns Data Model Transformation, Data Format Transformation, Protocol Bridging. Use this whenever the user needs to expose a mainframe, ERP, or other legacy system as a service, integrate a system that only exchanges flat files or batch files, consolidate channel-specific legacy back ends (branch, call center, web) behind one service, map between two different data models or schemas, convert between data formats (XML/JSON/CSV/EDI/fixed-width), or connect services that speak different protocols (SOAP to REST, MQ to HTTP, gRPC to REST). Also trigger for legacy modernization and wrapping proprietary APIs.
---

# SOA: Legacy Encapsulation and Transformation

Most real service inventories sit on top of systems that weren't built for them. These patterns
**hide legacy quirks behind standardized contracts** and **translate across the remaining
differences**. Transformation should be the exception: every transformation adds latency, cost,
and a place where meaning can be lost. Prefer standardization (Canonical Schema/Protocol) inside
an inventory, and use transformation at the edges. Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Legacy encapsulation

| Pattern | Problem | Distilled rule |
|---|---|---|
| [Legacy Wrapper](https://patterns.arcitura.com/soa-patterns/design_patterns/legacy_wrapper) | Consumers tied to a legacy system's proprietary API or data model inherit its constraints | Put a **wrapper service with a standardized, non-proprietary contract** in front of the legacy system. Consumers couple to the wrapper, so the legacy system can be replaced later without breaking them. The wrapper is the right place for an anti-corruption layer. Keep it thin and resist putting new business logic in it. |
| [File Gateway](https://patterns.arcitura.com/soa-patterns/design_patterns/file_gateway) | Legacy systems can only exchange **flat files** (batch exports, fixed-width, CSV drops) | An **intermediary gateway** watches for files, parses them, and turns them into service calls or messages, and turns outbound data back into files. Handle partial files (wait for a completion marker or rename), duplicates (idempotency by file name plus checksum), and records that fail to parse (route them to an error area and report them). Expect batch latency. |
| [Multi-Channel Endpoint](https://patterns.arcitura.com/soa-patterns/design_patterns/multi_channel_endpoint) | **Legacy systems custom-built for each delivery channel** (branch, call center, web, kiosk) duplicate logic and create silos | An **intermediary service encapsulates the channel-specific legacy systems** behind one standardized contract that all channels use. It needs substantial workflow logic and may need orchestration middleware. It can become a bottleneck, so scale it like the shared service it is. |

## Transformation

| Pattern | Converts | Distilled rule |
|---|---|---|
| [Data Model Transformation](https://patterns.arcitura.com/soa-patterns/design_patterns/data_model_transformation) | **Structure and meaning**: one schema or entity model to another (e.g., `Client{name}` to `Customer{firstName,lastName}`) | Build explicit, tested mapping logic between the models. It's needed at domain-inventory boundaries and with external partners. Each mapping adds latency and is a place where data can be lost (fields with no equivalent), so document lossy mappings. Too many mappings is a sign you need a Canonical Schema. |
| [Data Format Transformation](https://patterns.arcitura.com/soa-patterns/design_patterns/data_format_transformation) | **Representation** of equivalent data: XML ↔ JSON ↔ CSV ↔ EDI ↔ fixed-width | Separate format conversion from model mapping. Different concerns, different components. Format conversion is usually mechanical and library-driven. Watch encodings (character sets, numeric/date formats, decimal precision). |
| [Protocol Bridging](https://patterns.arcitura.com/soa-patterns/design_patterns/protocol_bridging) | **Communication protocol**: SOAP ↔ REST, MQ ↔ HTTP, gRPC ↔ REST/JSON, file ↔ API | A bridging intermediary (gateway, broker, ESB, adapter) converts between protocols at runtime. Protocol semantics don't map 1:1 (sync vs async, transactions, delivery guarantees, streaming), so state which guarantees survive the bridge. |

## Choosing

1. Is the legacy system's API or model the problem? → **Legacy Wrapper**.
2. Does it speak only in files? → **File Gateway** (plus Legacy Wrapper for a service contract).
3. Is the same capability duplicated per channel? → **Multi-Channel Endpoint**.
4. Same protocol and format, but different schema? → **Data Model Transformation**.
5. Same data, different syntax? → **Data Format Transformation**.
6. Different transport or interaction style? → **Protocol Bridging**.

The three transformations are often layered: bridge the protocol, convert the format, then map the
model. Implement them as separate steps (pipes and filters), not one opaque blob of glue code.

## Related skills

- `eip-message-transformation`: Normalizer, Canonical Data Model, Content Enricher, Envelope Wrapper
- `integration-architecture-patterns`: Anti-Corruption Layer and Hybrid Integration
- `legacy-modernizer` (agent) and `migration-assistant`: broader modernization planning
