# ASVS 5.0.0 — V1–V5: Encoding, Validation, Web Frontend, API, Files

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`. Where a requirement says "documented", the rule is app-specific;
see the "Documented security decisions" section in `SKILL.md`.

## V1 Encoding and Sanitization

Prefer safe APIs (parameterized queries, auto-escaping templates). Otherwise encode, escape, or
sanitize at the point of use. Input validation is defense in depth here (see V2), not the fix.

### V1.1 Architecture
| ID | L | Requirement |
|---|---|---|
| 1.1.1 | 2 | Decode/unescape input into canonical form **once**, only when that encoding is expected, and **before** validation/sanitization, never after. |
| 1.1.2 | 2 | Encode/escape output as the **final step** before the target interpreter, or let the interpreter do it. Store data raw, not pre-encoded (avoids double encoding). |

### V1.2 Injection Prevention
| ID | L | Requirement |
|---|---|---|
| 1.2.1 | 1 | Output encoding matches the context: HTML element, HTML attribute, HTML comment, CSS, HTTP header field, XML. |
| 1.2.2 | 1 | Dynamically built URLs: encode untrusted parts per context (URL-encode / base64url for query or path). Allow only safe schemes (no `javascript:`, `data:`). |
| 1.2.3 | 1 | Encode/escape when dynamically building JavaScript or JSON (JS/JSON injection). |
| 1.2.4 | 1 | SQL, HQL, NoSQL, Cypher, etc. use parameterized queries / ORMs / entity frameworks, **including inside stored procedures**. |
| 1.2.5 | 1 | OS command injection: use parameterized OS calls (argument arrays, no shell) or contextual command-line encoding. |
| 1.2.6 | 2 | Protect against LDAP injection. |
| 1.2.7 | 2 | Protect against XPath injection with parameterized or precompiled queries. |
| 1.2.8 | 2 | LaTeX processors: no `--shell-escape`; allowlist commands. |
| 1.2.9 | 2 | Escape regex metacharacters when untrusted input is embedded in a regular expression. |
| 1.2.10 | 3 | CSV/formula injection: apply RFC 4180 §2.6–2.7 escaping. In CSV/XLS/XLSX/ODF exports, prefix `'` when a field starts with `=`, `+`, `-`, `@`, tab, or `\0`. |

> **Note:** parameterization does not cover identifiers. Table names, column names, and
> `ORDER BY` columns cannot be escaped. Map user choices to an allowlist of known identifiers.

### V1.3 Sanitization (when encoding isn't possible)
| ID | L | Requirement |
|---|---|---|
| 1.3.1 | 1 | Untrusted HTML (WYSIWYG editors, etc.) is sanitized with a well-known HTML sanitization library or framework feature. |
| 1.3.2 | 1 | Avoid `eval()` and dynamic code execution (including expression languages like SpEL). If unavoidable, sanitize user input first. |
| 1.3.3 | 2 | Data passed to a dangerous context is sanitized: only safe characters allowed, overlong input trimmed. |
| 1.3.4 | 2 | User-supplied SVG is limited to safe tags/attributes (no scripts, no `foreignObject`). |
| 1.3.5 | 2 | User-supplied scriptable/template content (Markdown, CSS, XSL, BBCode, etc.) is sanitized or disabled. |
| 1.3.6 | 2 | **SSRF:** validate outbound targets against an allowlist of protocols, domains, paths, and ports, and sanitize dangerous characters before calling. |
| 1.3.7 | 2 | **Template injection:** never build templates from untrusted input. If unavoidable, sanitize or strictly validate it. |
| 1.3.8 | 2 | JNDI: sanitize input used in lookups; configure JNDI securely. |
| 1.3.9 | 2 | Sanitize content before sending it to memcache. |
| 1.3.10 | 2 | Sanitize format strings that could resolve unexpectedly before processing. |
| 1.3.11 | 2 | Sanitize input passed to mail systems (SMTP/IMAP injection). |
| 1.3.12 | 3 | Regexes are free of exponential backtracking; untrusted input is constrained (ReDoS). |

### V1.4 Memory, String, and Unmanaged Code
| ID | L | Requirement |
|---|---|---|
| 1.4.1 | 2 | Use memory-safe string handling, safer memory copy, and pointer arithmetic to prevent stack/buffer/heap overflows. |
| 1.4.2 | 2 | Use sign, range, and input validation to prevent integer overflows. |
| 1.4.3 | 2 | Release memory and resources; null out references to freed memory (no dangling pointers or use-after-free). |

Compiler hardening (stack protection, ASLR, DEP, build breaking on unsafe ops) can help satisfy these.

### V1.5 Safe Deserialization
| ID | L | Requirement |
|---|---|---|
| 1.5.1 | 1 | XML parsers use a restrictive config. External entity resolution is disabled (XXE). |
| 1.5.2 | 2 | Deserializing untrusted data enforces an allowlist of object types or restricts client-defined types. Mechanisms documented as insecure (by the language or framework) are **never** used on untrusted input. |
| 1.5.3 | 3 | All parsers for the same data type (JSON, XML, URL) parse consistently and use the same character encoding, so parser differentials can't be exploited (JSON interop bugs, SSRF, RFI). |

## V2 Validation and Business Logic

### V2.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 2.1.1 | 1 | Document input validation rules for each data item's expected structure (card numbers, emails, phones, internal formats). |
| 2.1.2 | 2 | Document how to check logical/contextual consistency of combined items (e.g., suburb matches ZIP). |
| 2.1.3 | 2 | Document business logic limits and validations, both per-user and global. |

### V2.2 Input Validation
| ID | L | Requirement |
|---|---|---|
| 2.2.1 | 1 | Positive validation: an allowlist of values/patterns/ranges, or comparison against an expected structure with logical limits. **L1:** at least input that drives business/security decisions. **L2+:** all input. |
| 2.2.2 | 1 | Validation is enforced at a **trusted service layer**. Client-side validation is UX only, never a security control. |
| 2.2.3 | 2 | Combinations of related items are checked against predefined rules. |

Schema validation (JSON Schema, XSD) is often the most effective way to get full coverage of HTTP
APIs. Do not use DTD validation; disable framework DTD evaluation (XXE).

### V2.3 Business Logic Security
| ID | L | Requirement |
|---|---|---|
| 2.3.1 | 1 | Business flows for a user run only in the expected step order, without skipping steps. |
| 2.3.2 | 2 | Documented business logic limits are enforced. |
| 2.3.3 | 2 | Business operations are transactional: they succeed entirely or roll back to the previous correct state. |
| 2.3.4 | 2 | Locking prevents double-booking of limited resources (seats, delivery slots). |
| 2.3.5 | 3 | High-value flows (large transfers, contract approvals, classified access, safety overrides) require multi-user approval. |

### V2.4 Anti-automation
| ID | L | Requirement |
|---|---|---|
| 2.4.1 | 2 | Anti-automation controls stop excessive calls that could cause data exfiltration, garbage data, quota exhaustion, rate-limit breaches, DoS, or overuse of costly resources. |
| 2.4.2 | 3 | Business flows require realistic human timing (reject implausibly fast submissions). |

## V3 Web Frontend Security (not applicable to machine-to-machine)

### V3.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 3.1.1 | 3 | Document the browser security features the app requires (HTTPS, HSTS, CSP, etc.) and what happens when they are missing (warn or block). |

### V3.2 Unintended Content Interpretation
| ID | L | Requirement |
|---|---|---|
| 3.2.1 | 1 | Prevent rendering in the wrong context when an API response, uploaded file, or other resource is requested directly. Options: check `Sec-Fetch-*` headers, use CSP `sandbox`, or set `Content-Disposition: attachment`. |
| 3.2.2 | 1 | Text meant as text is rendered with safe APIs (`textContent`, `createTextNode`), not as HTML. |
| 3.2.3 | 3 | Avoid DOM clobbering: explicit variable declarations, strict type checks, no globals on `document`, namespace isolation. |

### V3.3 Cookie Setup
| ID | L | Requirement |
|---|---|---|
| 3.3.1 | 1 | Cookies have `Secure`. Names use the `__Host-` prefix, or `__Secure-` if not `__Host-`. |
| 3.3.2 | 2 | `SameSite` is set per cookie purpose (limits CSRF and UI redress). |
| 3.3.3 | 2 | Use the `__Host-` prefix unless the cookie is explicitly designed to be shared with other hosts. |
| 3.3.4 | 2 | Cookies not needed by JS (e.g., session tokens) are `HttpOnly`, and that value is transferred **only** via `Set-Cookie`. |
| 3.3.5 | 3 | Cookie name + value ≤ 4096 bytes (oversized cookies are silently dropped). |

### V3.4 Browser Security Mechanism Headers
| ID | L | Requirement |
|---|---|---|
| 3.4.1 | 1 | `Strict-Transport-Security` on **all** responses, `max-age` ≥ 1 year. **L2+:** `includeSubDomains`. |
| 3.4.2 | 1 | CORS `Access-Control-Allow-Origin` is fixed, or the request `Origin` is validated against an allowlist. `*` only on responses with no sensitive data. |
| 3.4.3 | 2 | `Content-Security-Policy` at minimum: a global policy with `object-src 'none'` and `base-uri 'none'`, plus an allowlist or nonces/hashes. **L3:** per-response policy with nonces/hashes. |
| 3.4.4 | 2 | `X-Content-Type-Options: nosniff` on all responses. |
| 3.4.5 | 2 | A `Referrer-Policy` (header or HTML attribute) keeps paths, queries, and internal hostnames from leaking to third parties. |
| 3.4.6 | 2 | CSP `frame-ancestors` on every response: not embeddable by default, embedding allowed only where needed. `X-Frame-Options` is obsolete; don't rely on it. |
| 3.4.7 | 3 | CSP specifies a violation reporting location. |
| 3.4.8 | 3 | Document responses (`text/html`) send `Cross-Origin-Opener-Policy: same-origin` (or `same-origin-allow-popups`). |

### V3.5 Browser Origin Separation (CSRF and friends)
| ID | L | Requirement |
|---|---|---|
| 3.5.1 | 1 | If not relying on CORS preflight, sensitive requests carry anti-forgery tokens or a non-CORS-safelisted custom header that the server validates. |
| 3.5.2 | 1 | If relying on CORS preflight, sensitive functionality can't be reached by a request that skips preflight (check `Origin` and `Content-Type`, or require a custom header). |
| 3.5.3 | 1 | Sensitive functionality uses POST/PUT/PATCH/DELETE, never GET/HEAD/OPTIONS. Alternatively, strictly validate `Sec-Fetch-*`. |
| 3.5.4 | 2 | Separate applications live on separate hostnames (same-origin policy, cookie scoping). |
| 3.5.5 | 2 | `postMessage` handlers discard messages from untrusted origins or with invalid syntax. |
| 3.5.6 | 3 | No JSONP anywhere (XSSI). |
| 3.5.7 | 3 | No authorization-requiring data in script responses (XSSI). |
| 3.5.8 | 3 | Authenticated resources (images, video, scripts) can be loaded or embedded only when intended: validate `Sec-Fetch-*` or set a restrictive `Cross-Origin-Resource-Policy`. |

### V3.6 External Resource Integrity
| ID | L | Requirement |
|---|---|---|
| 3.6.1 | 3 | Externally hosted assets (CDN JS/CSS/fonts) only if static, versioned, and protected by Subresource Integrity. Otherwise, a documented decision per resource. |

### V3.7 Other Browser Considerations
| ID | L | Requirement |
|---|---|---|
| 3.7.1 | 2 | No unsupported or insecure client tech (NSAPI plugins, Flash, Shockwave, ActiveX, Silverlight, NaCl, Java applets). |
| 3.7.2 | 2 | Automatic redirects to hosts the app doesn't control go only to allowlisted destinations (open redirect). |
| 3.7.3 | 3 | Users are notified, with an option to cancel, when redirected off-app. |
| 3.7.4 | 3 | The top-level domain is on the HSTS preload list. |
| 3.7.5 | 3 | The app behaves as documented (warn or block) when the browser lacks required security features. |

## V4 API and Web Service (V6/V7/V2 still apply to APIs)

### V4.1 Generic Web Service Security
| ID | L | Requirement |
|---|---|---|
| 4.1.1 | 1 | Every response with a body has a `Content-Type` matching the content, with a safe `charset`. |
| 4.1.2 | 2 | Only user-facing browser endpoints auto-redirect HTTP→HTTPS. APIs don't, so clients mistakenly using plaintext fail loudly instead of silently leaking. |
| 4.1.3 | 2 | Headers set by intermediaries (`X-Real-IP`, `X-Forwarded-*`, `X-User-ID`) can't be overridden by end users. |
| 4.1.4 | 3 | Only explicitly supported HTTP methods (including OPTIONS for preflight) are accepted; others are blocked. |
| 4.1.5 | 3 | Per-message digital signatures for highly sensitive transactions or those crossing many systems. |

### V4.2 HTTP Message Structure Validation
| ID | L | Requirement |
|---|---|---|
| 4.2.1 | 2 | All components (LBs, firewalls, app servers) determine message boundaries consistently (request smuggling). HTTP/1.x: `Transfer-Encoding` overrides `Content-Length`. HTTP/2/3: `Content-Length` must match the DATA frames. |
| 4.2.2 | 3 | Generated messages never have a `Content-Length` that conflicts with protocol framing. |
| 4.2.3 | 3 | Never send or accept HTTP/2/3 messages with connection-specific headers like `Transfer-Encoding`. |
| 4.2.4 | 3 | Reject HTTP/2/3 header names or values containing CR, LF, or CRLF. |
| 4.2.5 | 3 | Outbound requests don't produce URIs or headers (e.g., `Cookie`, `Authorization`) too long for the receiver (self-inflicted DoS). |

### V4.3 GraphQL
| ID | L | Requirement |
|---|---|---|
| 4.3.1 | 2 | Query allowlist, depth limit, amount limit, or cost analysis prevents expensive nested-query DoS. |
| 4.3.2 | 2 | Introspection is disabled in production unless the API is meant for third parties. |

ASVS also recommends (Appendix D) putting GraphQL authorization in the business logic layer, not in
individual resolvers.

### V4.4 WebSocket
| ID | L | Requirement |
|---|---|---|
| 4.4.1 | 1 | WSS (WebSocket over TLS) for all WebSocket connections. |
| 4.4.2 | 2 | The `Origin` header is checked against an allowlist during the handshake. |
| 4.4.3 | 2 | If standard session management can't be used, dedicated tokens meet the V7 session requirements. |
| 4.4.4 | 2 | Dedicated WebSocket tokens are obtained or validated through the already-authenticated HTTPS session. |

## V5 File Handling

### V5.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 5.1.1 | 2 | For each upload feature, document permitted types, extensions, max size (including **unpacked** size), how files are made safe for download/processing, and behavior when a malicious file is detected. |

### V5.2 File Upload and Content
| ID | L | Requirement |
|---|---|---|
| 5.2.1 | 1 | Accept only files of a size the app can process without performance loss or DoS. |
| 5.2.2 | 1 | Extension matches an expected one **and** content matches the extension (magic bytes, image re-writing, content-validation libraries), including files inside archives. **L1:** files driving business/security decisions. **L2+:** all files. |
| 5.2.3 | 2 | Compressed files (zip, gz, docx, odt) are checked against max uncompressed size and max file count **before** decompressing. |
| 5.2.4 | 3 | Per-user storage quota and file-count limit. |
| 5.2.5 | 3 | Reject archives containing symlinks unless required (then allowlist targets). |
| 5.2.6 | 3 | Reject images above a maximum pixel size (pixel flood). |

### V5.3 File Storage
| ID | L | Requirement |
|---|---|---|
| 5.3.1 | 1 | Files uploaded or generated from untrusted input and stored in a public folder are never executed as server-side code. |
| 5.3.2 | 1 | File paths are built from internally generated or trusted data. If user filenames/metadata must be used, strictly validate and sanitize them (path traversal, LFI/RFI, SSRF). |
| 5.3.3 | 3 | Server-side processing (e.g., decompression) ignores user-provided path info (zip slip). |

### V5.4 File Download
| ID | L | Requirement |
|---|---|---|
| 5.4.1 | 2 | User-submitted filenames (including in JSON or URL params) are validated or ignored, and the response sets the filename in `Content-Disposition`. |
| 5.4.2 | 2 | Served filenames (headers, email attachments) are encoded or sanitized (RFC 6266). |
| 5.4.3 | 2 | Files from untrusted sources are antivirus-scanned before serving. |
