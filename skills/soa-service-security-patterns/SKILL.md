---
name: soa-service-security-patterns
description: Secure services and service-to-service interactions using SOA security patterns (Thomas Erl / Arcitura) — service security patterns Exception Shielding, Message Screening, Service Perimeter Guard, Trusted Subsystem; and interaction security patterns Direct Authentication, Brokered Authentication, Data Confidentiality, Data Origin Authentication. Use this whenever the user asks how services should authenticate callers (shared secrets vs token issuer/STS/OAuth), whether a service should call its database with the user's identity or its own, how to stop error responses leaking internals, how to validate incoming messages for malicious content (injection, XML bombs, oversize payloads), how to expose internal services to external consumers safely (DMZ, gateway), message-level encryption vs TLS, or signing messages to prove origin and integrity. Also trigger for zero-trust service design and microservice security reviews.
---

# SOA: Service Security Patterns

Services multiply the attack surface: every contract is an entry point, and every hop is a place
to intercept, impersonate, or tamper. These patterns split the job into **protecting a service**
(what it accepts and what it reveals) and **protecting interactions** (who is calling, whether the
data stays secret, and whether it's genuine). Source: [Arcitura SOA Patterns](https://patterns.arcitura.com/soa-patterns).

## Protecting the service itself

| Pattern | Threat | Distilled rule |
|---|---|---|
| [Exception Shielding](https://patterns.arcitura.com/soa-patterns/design_patterns/exception_shielding) | Error responses leak stack traces, SQL, hostnames, and library versions that help attackers | **Sanitize exceptions before they leave the service.** Return a generic, structured error (Error Report / RFC 9457) with a correlation ID, and log the full detail internally under that ID. Apply it centrally (a middleware or exception mapper) so nobody can forget it. |
| [Message Screening](https://patterns.arcitura.com/soa-patterns/design_patterns/message_screening) | Malicious payloads: injection (SQL, command, XPath, LDAP), XML external entities and billion-laughs bombs, oversized or deeply nested JSON, malformed binary | **Screen every incoming message before processing**: size and depth limits, strict schema validation, parser hardening (disable XXE/DTDs), and allow-lists for content types. It costs some performance, so put coarse checks at the gateway and precise checks in the service. |
| [Service Perimeter Guard](https://patterns.arcitura.com/soa-patterns/design_patterns/service_perimeter_guard) | Internal services exposed directly to external consumers | Put an **intermediary service at the network perimeter** (DMZ) as the only thing external consumers can reach. It forwards vetted requests to internal services, so the internal ones are never directly exposed. Examples: API gateway, WAF + reverse proxy, Inventory Endpoint. It's a single point of failure and a bottleneck, so deploy it redundantly. |
| [Trusted Subsystem](https://patterns.arcitura.com/soa-patterns/design_patterns/trusted_subsystem) | Consumers given direct credentials to back-end resources (DB, legacy system) can misuse them | The **service accesses resources with its own identity**, not the consumer's. Consumers are authorized *by the service* and never hold back-end credentials. Use managed/workload identities, not stored secrets. **Trade-off:** the service is fully trusted, so a compromised service has full access, and per-user auditing at the resource level is lost. Log the original caller's identity, and consider passing the user context through (on-behalf-of token flows) where the resource must enforce per-user rules. |

## Protecting interactions

### Who is calling? Authentication

| Pattern | How it works | Choose it when | Trade-offs |
|---|---|---|---|
| [Direct Authentication](https://patterns.arcitura.com/soa-patterns/design_patterns/direct_authentication) | The **service itself checks the consumer's credentials** (username/password, API key + secret, client cert) against its own or a shared identity store | Few consumers and services, and a simple trust relationship | Every service handles credentials, consumers re-authenticate everywhere, and credential sprawl |
| [Brokered Authentication](https://patterns.arcitura.com/soa-patterns/design_patterns/brokered_authentication) | An **authentication broker** (STS, Kerberos KDC, OAuth 2.0/OIDC authorization server) checks the consumer once and **issues a token**, which services validate | Many services and consumers, SSO, federation across trust domains | The broker is critical infrastructure. Validate tokens properly: signature, issuer, audience, expiry. Short-lived tokens plus refresh |

**Modern default:** brokered authentication with OAuth 2.0/OIDC (user and client tokens) plus
mTLS or workload identity (SPIFFE, cloud managed identities) for service-to-service calls. Direct
authentication only for small, closed integrations.

### Is the data secret? Is it genuine?

| Pattern | Guarantees | Distilled rule |
|---|---|---|
| [Data Confidentiality](https://patterns.arcitura.com/soa-patterns/design_patterns/data_confidentiality) | Only the intended recipient can read the data | **TLS protects one hop. Message-level encryption protects end to end**, through intermediaries, queues, and storage. Use message-level (or field-level) encryption (JWE, envelope encryption with KMS) when messages pass through brokers or intermediaries you don't fully trust, or are stored at rest in transit systems. It costs processing and key management. |
| [Data Origin Authentication](https://patterns.arcitura.com/soa-patterns/design_patterns/data_origin_authentication) | The message **came from who it claims** and **wasn't changed** | **Sign the message** (JWS, HMAC with a shared key, X.509 signatures, webhook signature headers). Receivers verify the signature *and* a timestamp/nonce to stop replay. Asymmetric signatures also give non-repudiation. Shared-key HMAC doesn't. |

TLS alone doesn't give you origin authentication of the *message* once it leaves the connection,
e.g., a webhook replayed later, or a message relayed through a queue. When that matters, sign it.

## Service security review checklist

1. Do all external requests enter through a perimeter guard (gateway/WAF), with internal services unreachable from outside?
2. Is every inbound message screened: size and depth limits, schema validation, hardened parsers?
3. Are errors shielded, with full details only in internal logs, keyed by correlation ID?
4. Is authentication brokered (tokens), with full token validation in every service?
5. Do services reach their resources with their own managed identity, while logging the original caller?
6. Is sensitive data encrypted end to end where it crosses intermediaries, and are important messages signed with replay protection?

## Related skills

- `map-message-structure-and-representation`: API Key and Error Report payload design
- `api-security-audit` / `security-auditor` (agents): hands-on API security reviews
- `azure-security-architecture`, `aws-iam-security-fundamentals`: platform-specific identity implementation
