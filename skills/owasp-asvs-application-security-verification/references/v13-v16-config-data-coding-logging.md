# ASVS 5.0.0 — V13–V16: Configuration, Data Protection, Secure Coding, Logging and Errors

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`.

## V13 Configuration (the default config must be safe on the Internet)

### V13.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 13.1.1 | 2 | Document all communication needs, including external services relied on and any case where a user can supply an external location the app will connect to. |
| 13.1.2 | 3 | For each service, document max concurrent connections (pool limits) and behavior at the limit, including fallback/recovery. |
| 13.1.3 | 3 | For each external resource (DBs, file handles, threads, HTTP), document the resource-management strategy: release, timeouts, failure handling, retry limits, delays, back-off. Synchronous HTTP: short timeouts and no retries or strictly limited ones (avoid cascading exhaustion). |
| 13.1.4 | 3 | Document the security-critical secrets and a rotation schedule based on the threat model. |

### V13.2 Backend Communication Configuration
| ID | L | Requirement |
|---|---|---|
| 13.2.1 | 2 | Backend component communication outside the user session mechanism (APIs, middleware, data layers) is authenticated with individual service accounts, short-term tokens, or certificates. No unchanging passwords, API keys, or shared privileged accounts. |
| 13.2.2 | 2 | Backend communication uses least-privilege accounts. |
| 13.2.3 | 2 | No default credentials (root/root, admin/admin) for service authentication. |
| 13.2.4 | 2 | An **allowlist** defines the external resources/systems the app may talk to (outbound requests, data loads, file access), at app, web server, or firewall layer. |
| 13.2.5 | 2 | The web/app server itself is configured with an allowlist of resources it may request or load from. |
| 13.2.6 | 3 | Each connection follows its documented config (max parallel connections, behavior at max, timeouts, retries). |

### V13.3 Secret Management
| ID | L | Requirement |
|---|---|---|
| 13.3.1 | 2 | A secrets manager (key vault) creates, stores, controls access to, and destroys backend secrets: passwords, keys, DB/third-party credentials, TOTP seeds, API keys. **No secrets in source code or build artifacts.** **L3:** hardware-backed (HSM). |
| 13.3.2 | 2 | Access to secrets follows least privilege. |
| 13.3.3 | 3 | All crypto operations happen inside an isolated security module (vault/HSM); key material never leaves it. |
| 13.3.4 | 3 | Secrets expire and rotate per the documentation. |

### V13.4 Unintended Information Leakage
| ID | L | Requirement |
|---|---|---|
| 13.4.1 | 1 | No source control metadata (`.git`, `.svn`) deployed, or it's inaccessible both externally and to the app. |
| 13.4.2 | 2 | Debug modes are disabled for all production components. |
| 13.4.3 | 2 | No directory listings unless intended. |
| 13.4.4 | 2 | HTTP `TRACE` is not supported in production. |
| 13.4.5 | 2 | Internal API docs (Swagger/OpenAPI UIs) and monitoring endpoints (`/actuator`, `/metrics`, `/debug`) aren't exposed unless intended. |
| 13.4.6 | 3 | No detailed backend version info exposed. |
| 13.4.7 | 3 | The web tier serves only specific file extensions (no stray configs, backups, source). |

## V14 Data Protection

### V14.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 14.1.1 | 2 | All sensitive data is identified and classified into protection levels, **including merely encoded data** (Base64, JWT payloads), taking applicable privacy regulations into account. |
| 14.1.2 | 2 | Each protection level has documented requirements: encryption, integrity, retention, logging rules, log access control, DB-level encryption, privacy-enhancing tech, other confidentiality needs. |

### V14.2 General Data Protection
| ID | L | Requirement |
|---|---|---|
| 14.2.1 | 1 | Sensitive data goes only in the HTTP body or headers, **never in the URL or query string** (API keys, session tokens). |
| 14.2.2 | 2 | Sensitive data isn't cached in server components (LBs, app caches), or is securely purged after use. |
| 14.2.3 | 2 | Defined sensitive data isn't sent to untrusted parties (e.g., analytics trackers). |
| 14.2.4 | 2 | The documented controls for each protection level are implemented. |
| 14.2.5 | 3 | Caches store only responses with the expected content type and no sensitive dynamic content. Non-existent paths return 404/302, not another valid file (**web cache deception**). |
| 14.2.6 | 3 | Return the minimum sensitive data (e.g., last 4 card digits). If full data is needed, mask it in the UI until explicitly revealed. |
| 14.2.7 | 3 | Retention classification: outdated or unnecessary sensitive data is deleted automatically or on schedule. |
| 14.2.8 | 3 | Metadata (EXIF, etc.) is stripped from user-submitted files unless the user consents to keeping it. |

### V14.3 Client-side Data Protection
| ID | L | Requirement |
|---|---|---|
| 14.3.1 | 1 | Authenticated data is cleared from client storage and the DOM when the client/session ends. `Clear-Site-Data` helps, but the client must also clean up when the server is unreachable. |
| 14.3.2 | 2 | `Cache-Control: no-store` (sufficient anti-caching headers) on sensitive responses. |
| 14.3.3 | 2 | No sensitive data in `localStorage`, `sessionStorage`, IndexedDB, or cookies, **except session tokens**. |

## V15 Secure Coding and Architecture

*Dangerous functionality* means deserialization of untrusted data, raw file/binary parsing,
dynamic code execution, and direct memory manipulation. A *risky component* is a third-party
library that is poorly maintained, unsupported, end-of-life, or has a history of serious vulns.

### V15.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 15.1.1 | 1 | Document **risk-based remediation timeframes** for vulnerable third-party components and for routine library updates. |
| 15.1.2 | 2 | Maintain an inventory (**SBOM**) of all third-party libraries, sourced from predefined, trusted, maintained repositories. |
| 15.1.3 | 2 | Document time-consuming or resource-heavy functionality and how availability is protected (async processing, queues, per-user and per-app parallelism limits), including how to avoid responses outlasting consumer timeouts. |
| 15.1.4 | 3 | Document which third-party libraries are "risky components". |
| 15.1.5 | 3 | Document where "dangerous functionality" is used. |

### V15.2 Security Architecture and Dependencies
| ID | L | Requirement |
|---|---|---|
| 15.2.1 | 1 | No component has breached its documented update/remediation timeframe. |
| 15.2.2 | 2 | Availability defenses for expensive functionality are implemented as documented. |
| 15.2.3 | 2 | Production contains only needed functionality: no test code, samples, or dev features. |
| 15.2.4 | 3 | All components and **transitive** dependencies come from the expected repository; no **dependency confusion** risk. |
| 15.2.5 | 3 | Extra isolation (sandboxing, encapsulation, containers, network segmentation) around dangerous functionality and risky components limits pivoting. |

### V15.3 Defensive Coding
| ID | L | Requirement |
|---|---|---|
| 15.3.1 | 1 | Return only the required subset of fields, never whole data objects (over-exposure). |
| 15.3.2 | 2 | Backend calls to external URLs don't follow redirects unless intended (SSRF via redirect). |
| 15.3.3 | 2 | **Mass assignment** countermeasures: allowed fields are limited per controller/action. |
| 15.3.4 | 2 | Proxies and middleware convey the original client IP via trusted fields the user can't forge, and app and web server use that value for logging and rate limiting (remembering IPs are still unreliable). |
| 15.3.5 | 2 | Strict types and strict equality/comparison (no **type juggling** or type confusion). |
| 15.3.6 | 2 | JavaScript avoids **prototype pollution** (e.g., `Map`/`Set` instead of object literals for keyed data). |
| 15.3.7 | 2 | Defenses against **HTTP parameter pollution**, especially where the framework merges query, body, cookie, and header params. |

### V15.4 Safe Concurrency
| ID | L | Requirement |
|---|---|---|
| 15.4.1 | 3 | Shared objects in multi-threaded code (caches, files, in-memory state) use thread-safe types and synchronization. |
| 15.4.2 | 3 | State checks and dependent actions are atomic (**TOCTOU**), e.g., file-exists-then-open or check-access-then-grant. |
| 15.4.3 | 3 | Locks are used consistently (no deadlock or livelock), with locking logic encapsulated so external code can't tamper with it. |
| 15.4.4 | 3 | Resource allocation is fair (thread pools, no starvation of low-priority work). |

## V16 Security Logging and Error Handling

Security logs are distinct from debug/perf logs: high-signal, structured, SIEM-ready. Alerting and
correlation rules are out of ASVS scope.

### V16.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 16.1.1 | 2 | A log inventory covers each stack layer: events logged, formats, storage location, usage, access control, retention. |

### V16.2 General Logging
| ID | L | Requirement |
|---|---|---|
| 16.2.1 | 2 | Each entry has the metadata to reconstruct a timeline: **when, where, who, what**. |
| 16.2.2 | 2 | Clocks are synchronized; timestamps are **UTC** (recommended) or carry an explicit offset. |
| 16.2.3 | 2 | Logs go only to the destinations documented in the inventory. |
| 16.2.4 | 2 | Logs are parseable and correlatable by the log processor (common format). |
| 16.2.5 | 2 | Logging follows data protection levels: never log credentials or payment details; session tokens only hashed or masked. |

### V16.3 Security Events
| ID | L | Requirement |
|---|---|---|
| 16.3.1 | 2 | Log all authentication operations, successes and failures, with auth type/factors. |
| 16.3.2 | 2 | Log failed authorization attempts. **L3:** log all authorization decisions, including sensitive-data access (without the data itself). |
| 16.3.3 | 2 | Log documented security events and **bypass attempts** (input validation, business logic, anti-automation). |
| 16.3.4 | 2 | Log unexpected errors and security control failures (e.g., backend TLS failures). |

### V16.4 Log Protection
| ID | L | Requirement |
|---|---|---|
| 16.4.1 | 2 | Encode logged data to prevent **log injection** (CR/LF, control chars, forged entries). |
| 16.4.2 | 2 | Logs are protected from unauthorized access and modification. |
| 16.4.3 | 2 | Logs ship securely to a logically separate system, so a breach of the app doesn't compromise them. |

### V16.5 Error Handling
| ID | L | Requirement |
|---|---|---|
| 16.5.1 | 2 | Unexpected or security-sensitive errors return a **generic message**: no stack traces, queries, keys, or tokens. |
| 16.5.2 | 2 | The app stays secure when external resources fail (circuit breakers, graceful degradation). |
| 16.5.3 | 2 | **Fail closed:** an exception never results in fail-open behavior (e.g., processing a transaction despite a validation error). |
| 16.5.4 | 3 | A last-resort handler catches all unhandled exceptions, preserving log details and keeping the process alive. In languages without exceptions (Go, Swift), use the idiomatic equivalent. |
