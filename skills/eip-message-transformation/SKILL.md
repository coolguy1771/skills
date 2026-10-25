---
name: eip-message-transformation
description: Apply the Enterprise Integration Patterns for message transformation — Envelope Wrapper, Content Enricher, Content Filter, Claim Check, Normalizer, and Canonical Data Model (plus the base Message Translator). Use this whenever the user needs to convert between data formats across systems, add missing data to a message before forwarding, strip a large or sensitive payload down to what a consumer needs, handle large payloads in a queue or event bus ("message too large," "store the blob and pass a reference"), accept the same business document from many partners in different formats, or decide whether to adopt an enterprise canonical data model. Also trigger for ETL/iPaaS mapping design, anti-corruption mapping in messaging, and reducing point-to-point translator sprawl.
---

# EIP: Message Transformation

Every integration eventually has to answer "their format isn't our format." These patterns are
specialized [Message Translators](https://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageTranslator.html).
Each answers a different version of that question. Pick by **what's wrong with the message**:

| What's wrong | Pattern |
|---|---|
| The transport needs headers, encryption, or a format the app doesn't produce | Envelope Wrapper |
| Data the receiver needs is missing | Content Enricher |
| The message has too much data (size, sensitivity, complexity) | Content Filter or Claim Check |
| Same meaning, many incoming formats | Normalizer |
| N systems × M formats is exploding | Canonical Data Model |

## Patterns

### [Envelope Wrapper](https://www.enterpriseintegrationpatterns.com/patterns/messaging/EnvelopeWrapper.html)
**Problem:** existing systems need to take part in an exchange that requires particular header
fields, encryption, or an envelope format.
**Solution:** a wrapper puts the app's data inside an envelope that meets the messaging
infrastructure's requirements. An unwrapper removes it at the other end. Envelopes can nest (an
app message inside a SOAP envelope inside a transport frame), like a TCP/IP stack.
**Rule:** unwrap only at the boundary that wrapped it. Intermediaries shouldn't open envelopes
they don't own. That's how end-to-end encryption and signatures stay intact.

### [Content Enricher](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DataEnricher.html)
**Problem:** the receiver needs data the originator doesn't have.
**Solution:** an enricher looks up the missing data from a resource (database, service, cache,
computation) and adds it to the message.
**Rules:** enrich as late as possible, close to the consumer that needs the data, so the shared
message doesn't get bloated. Decide what happens when the lookup fails (block, enrich partially,
or route to error). The enricher's dependency on that resource now affects your throughput and
availability, so cache read-mostly reference data.

### [Content Filter](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentFilter.html)
**Problem:** a large message is hard to handle when you need only a few fields.
**Solution:** remove unneeded elements, and optionally flatten deep hierarchies into a simpler
structure.
**Uses:** reducing payloads for constrained consumers, **stripping PII or other sensitive fields**
before data crosses a trust boundary, and simplifying a vendor's verbose format.
Not the same as a *Message Filter*, which drops whole messages. A Content Filter removes *fields*.

### [Claim Check](https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html)
**Problem:** reduce the amount of data sent through the system without losing information.
**Solution:** store the payload in a persistent store (blob storage, S3, database), send a
**claim check** (a reference/key) in the message, and let the consumers that need the data
retrieve it.
**Rules:**
- Default to it for payloads beyond a few hundred KB, even if the broker's limit technically
  allows more. Brokers are bad at storing data in bulk.
- Decide the **lifecycle** of the stored data: who deletes it, and when (TTL, after the last
  consumer, retention policy). Orphaned blobs are the classic failure.
- Secure the store separately. Use short-lived signed URLs or identity-based access, not
  long-lived keys embedded in messages.
- Also useful for hiding sensitive data from intermediaries that only route on headers.

### [Normalizer](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Normalizer.html)
**Problem:** messages that mean the same thing arrive in different formats (purchase orders from
partners in EDI, CSV, JSON, XML...).
**Solution:** a router detects each message's format, sends it to a format-specific translator,
and every translator outputs **one common format**.
**Rule:** format detection should use an explicit Format Indicator or source channel where
possible. Sniffing content is fragile.

### [Canonical Data Model](https://www.enterpriseintegrationpatterns.com/patterns/messaging/CanonicalDataModel.html)
**Problem:** minimize dependencies when integrating apps that use different data formats.
**Solution:** define a data model independent of any single app. Each app translates only to and
from the canonical model, so translators grow as **2N instead of N×(N−1)**.
**Trade-offs:**
- Pays off when many systems exchange the same core entities (customer, order, product).
  Overkill for two or three integrations.
- Enterprise-wide canonical models tend to become huge, slow-moving committee artifacts. Bounded,
  per-domain canonical models (DDD's "published language") usually work better than one global
  schema.
- It adds translation steps (latency, and some fidelity loss where a system's richer data doesn't
  fit). Version the canonical model with a Format Indicator from day one.

## Design guidance

- **Keep transformation in thin, stateless filters.** Mixing transformation with routing or
  business rules in the same component makes all of them hard to change.
- **Test mappings like code**: golden-file tests per source format, plus property tests for
  round-trips where both directions exist.
- **Lossy transformations must be explicit.** Document which fields are dropped (Content Filter)
  or defaulted, so nobody downstream assumes they're there.

## Related skills

- `eip-messaging-channels-and-infrastructure`: Message Translator and Datatype Channel basics
- `integration-architecture-patterns`: Anti-Corruption Layer, the domain-model version of translation
- `soa-legacy-encapsulation-and-transformation`: Data Model / Data Format Transformation and Protocol Bridging in SOA terms
- `soa-service-inventory-design`: Canonical Schema, the SOA counterpart to Canonical Data Model
