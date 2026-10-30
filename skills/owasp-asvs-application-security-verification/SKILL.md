---
name: owasp-asvs-application-security-verification
description: Apply OWASP ASVS 5.0.0 (345 verifiable requirements, 17 chapters, levels L1/L2/L3) to design, build, review, or audit web apps and APIs. Covers injection/encoding, input validation, business logic, browser headers/cookies/CSRF/CORS, API/GraphQL/WebSocket, file upload, authentication/MFA/passwords, sessions, authorization/IDOR, JWTs, OAuth/OIDC, crypto algorithms, TLS, secrets/config, data protection, dependencies, logging/errors, WebRTC. Use when asked to "follow security best practices", "make this secure", "harden" an app or endpoint, do a security review, check against ASVS/OWASP, pick an ASVS level, write security requirements, choose headers/CSP/cookie flags or crypto parameters, or when implementing login, password reset, MFA, uploads, JWT validation, OAuth clients/servers, or security logging. Also for ASVS compliance reports and mapping findings to ASVS IDs.
---

# OWASP ASVS 5.0.0: Application Security Verification

The ASVS is a list of **verifiable, outcome-focused security requirements** for web apps and
APIs: about 345 requirements in 17 chapters, each a pass/fail statement ("Verify that…"). It says
*what must be true*, not *how* to implement it in a given stack; the OWASP Cheat Sheets cover
the how. Use it three ways: as **design guidance** when building, as a **secure-coding
checklist** when reviewing, and as a **verification standard** when auditing.

Cite requirements as `v5.0.0-<chapter>.<section>.<req>`, e.g. `v5.0.0-1.2.5`. IDs changed
completely from 4.x, so always include the version.

## Step 1: Pick the target level

Each level includes everything below it. Levels reflect risk reduction versus implementation
effort, not "how testable" a control is.

| Level | Share | What it is | Who should target it |
|---|---|---|---|
| **L1** | 70 reqs (~20%) | First-layer defenses against common attacks that need no preconditions. Password hygiene matters most here. | Bare minimum for anything internet-facing; the starting point for adoption. |
| **L2** | +183 reqs (L1+L2 ≈ 70%) | Comprehensive standard practice. Less common attacks, more involved protections. **MFA is required.** | **Default for most applications**, and anything handling sensitive or personal data. |
| **L3** | +92 reqs (100%) | Defense in depth and hard-to-implement controls. **Hardware-backed, phishing-resistant auth**, HSM-backed secrets, sender-constrained tokens. | High-assurance apps: banking, healthcare, critical infrastructure, high-value targets. |

If the user hasn't said, **assume L2** and say so. Apply L3 items where a specific risk justifies
it (e.g., a payments flow inside an L2 app). Some requirements tighten at higher levels inside
the same ID (e.g., `3.4.1` HSTS adds `includeSubDomains` at L2; `10.4.3` code lifetime drops to 1
min at L3). The reference tables call these out.

## Step 2: Scope the chapters

Drop chapters that don't apply. An M2M API skips V3; no OAuth means skip V10; no WebRTC means
skip V17. Then load **only** the reference file(s) for the chapters the task touches:

| Chapter | Applies when the app… | Reference file |
|---|---|---|
| V1 Encoding & Sanitization | Builds queries, commands, HTML, URLs, templates; parses XML/serialized data | `references/v01-v05-input-frontend-api-files.md` |
| V2 Validation & Business Logic | Accepts any input; has multi-step flows, limits, scarce resources | same |
| V3 Web Frontend | Is rendered in a browser (cookies, headers, CSRF, CORS, redirects) | same |
| V4 API & Web Service | Exposes HTTP APIs, GraphQL, WebSockets; sits behind proxies | same |
| V5 File Handling | Accepts uploads, serves downloads, unpacks archives | same |
| V6 Authentication | Has logins, passwords, MFA, recovery, or federates to an IdP | `references/v06-v08-authn-session-authz.md` |
| V7 Session Management | Keeps users logged in (cookies, tokens) | same |
| V8 Authorization | Has roles, ownership, tenants, admin functions (i.e., almost always) | same |
| V9 Self-contained Tokens | Issues or accepts JWTs, SAML assertions, signed tokens | `references/v09-v10-tokens-oauth-oidc.md` |
| V10 OAuth & OIDC | Is an OAuth client/RP, resource server, AS/OP, or does SSO | same |
| V11 Cryptography | Encrypts, hashes, signs, generates random secrets, stores passwords | `references/v11-v12-crypto-and-tls.md` (includes the approved-algorithm tables) |
| V12 Secure Communication | Talks over a network (always) | same |
| V13 Configuration | Is deployed anywhere, has secrets, makes outbound calls | `references/v13-v16-config-data-coding-logging.md` |
| V14 Data Protection | Handles PII, credentials, financial/health data, or anything sensitive | same |
| V15 Secure Coding & Architecture | Has dependencies, JS objects from user input, concurrency, expensive operations | same |
| V16 Logging & Error Handling | Logs anything or returns errors (always) | same |
| V17 WebRTC | Runs its own TURN, media, or signaling servers | `references/v17-webrtc.md` |

Every reference row is `| ID | L | Requirement |`, so you can filter by level (e.g., find all L1
rows) and grep by ID or keyword.

## Step 3: Apply it in the right mode

### Building or designing a feature
1. Identify the chapters the feature touches (login form → V6, V7, V3.3, V16.3, V2.4, V11.4).
2. Load those references and implement every requirement at the target level. Prefer
   well-vetted framework and library mechanisms over hand-rolled controls (centralized, simple,
   reusable). Use **one** authorization chokepoint that every request passes through.
3. Where a requirement starts with "documentation defines…", don't invent the rule silently. See
   *Documented security decisions* below.
4. Turn relevant requirements into automated tests (abuse cases: enumeration, brute force,
   IDOR, injection payloads, oversize input, step skipping). "Testable using automation ≠
   running an off-the-shelf scanner."

### Reviewing code or a PR
1. Map the diff to chapters, then walk the matching rows at the target level.
2. Report each gap as a finding tied to an ASVS ID (format below). Separate **confirmed
   violations** from **can't-verify-from-code** items (often documentation or infrastructure
   requirements). Don't mark those as failures; list them as open questions.
3. SAST/DAST catch some L1 encoding issues but cannot verify business logic or access control.
   Read the code paths yourself.

### Verifying or auditing an application (ASVS assessment)
A verification report is **not** "by exception" like a pentest report. It must include:
- **Scope and target level**, stated as what was *included*, with rationale for any exclusions.
- **Every requirement checked**, each marked Pass / Fail / N/A (N/A needs a reason, e.g.,
  "stateless API, no browser sessions").
- Exceptions found, with remediation guidance, and the testing methods used (repeatable).
- Evidence (code refs, requests/responses, config excerpts).

Black-box testing alone is explicitly discouraged, even for L1. Use documentation- or
source-led (hybrid) testing. Nobody can claim "OWASP-certified ASVS compliance": OWASP certifies
no one.

## Documented security decisions

ASVS 5.0 deliberately does not hard-code app-specific rules (allowed file types, session
timeouts, authorization matrix, validation rules, rate limits). Instead, each chapter's first
section (`x.1.*`) requires that the decision be **documented**, and a later requirement requires
the implementation to **match the documentation**. These are two separate checks.

When you hit one of these:
- If the project already records the decision (a security doc, ADR, config, or shared library
  that encodes it), implement or verify against it.
- If not, **propose a concrete, sensible default, label it as a decision that needs an owner,
  and record it** (ADR, `SECURITY.md`, or code comments next to the central config). Don't
  silently pick a value and move on.

Key documentation requirements: 2.1.1–2.1.3 (validation rules, business limits), 5.1.1 (upload
types/sizes), 6.1.1 (brute-force controls, lockout abuse), 6.1.3 (all auth pathways), 7.1.1–7.1.3
(session timeouts, concurrency, SSO), 8.1.1–8.1.4 (authorization rules), 11.1.1–11.1.2 (key
policy, crypto inventory), 13.1.x (communication needs, connection limits, secret rotation),
14.1.1–14.1.2 (data classification), 15.1.x (patch SLAs, SBOM, expensive operations, risky
components), 16.1.1 (log inventory).

## The non-negotiables (L1 core, plus the L2 items most often missed)

These are the rules to apply reflexively in any code you write or review. IDs point to the full
text.

**Injection and output**
- Parameterize every query (SQL/NoSQL/HQL/Cypher), stored procedures included (1.2.4). Map
  dynamic identifiers like `ORDER BY` columns to an allowlist; they can't be escaped.
- No shell string building; pass argument arrays (1.2.5). No `eval` or expression languages on
  user input (1.3.2). Never build templates from user input (1.3.7).
- Encode for the exact output context (HTML body/attribute/JS/CSS/URL/header) as the last step,
  and store raw data, not encoded (1.1.2, 1.2.1–1.2.3). Allowlist URL schemes: no
  `javascript:`/`data:` (1.2.2). Render text with `textContent` (3.2.2).
- Sanitize rich HTML with a vetted library such as DOMPurify (1.3.1). Disable XML external
  entities and DTDs (1.5.1). Never feed untrusted data to deserializers documented as unsafe
  (e.g., Java native serialization, Python `pickle`, .NET `BinaryFormatter`, YAML full loaders)
  (1.5.2).
- Outbound requests built from user input: allowlist protocol, host, port, and path (1.3.6,
  5.3.2); don't follow redirects by default (15.3.2); server-level egress allowlist (13.2.4–13.2.5).

**Input and business logic**
- Positive, server-side validation; client-side is UX only (2.2.1, 2.2.2).
- Enforce step order in multi-step flows (2.3.1); make them transactional, with locking for scarce
  resources (2.3.3, 2.3.4); add anti-automation and rate limits on costly or abusable functions
  (2.4.1).

**Authorization (the #1 real-world failure)**
- Check function-level **and** object-level access on every request, server-side (8.2.1, 8.2.2,
  8.3.1). "User can reach this endpoint" ≠ "user may touch this record". Scope queries by
  owner/tenant (8.4.1).
- Return only the fields the caller may see (15.3.1, 8.2.3) and accept only the fields the
  action allows, with no mass assignment (15.3.3).

**Authentication**
- Passwords: min 8 (15 recommended), allow ≥ 64, **no composition rules, no forced rotation**,
  allow paste and password managers, block the top-3000 and breached passwords, never
  truncate or case-fold (6.2.x).
- Store passwords with argon2id / scrypt / bcrypt / PBKDF2 at the Appendix C parameters (11.4.2).
- No default accounts (6.3.2); no secret questions or password hints (6.4.2); brute-force and
  credential-stuffing controls that can't be abused to lock users out (6.1.1, 6.3.1).
- L2: **MFA required** (6.3.3); reset must not bypass MFA (6.4.3); re-authenticate before
  changing email, phone, or MFA (7.5.1); no user enumeration via message, status, or timing
  (6.3.8, L3, but cheap, so do it anyway).
- Email is never an authentication factor. SMS only with a stronger alternative offered, and not
  at L3 (6.3.6, 6.6.1).

**Sessions and cookies**
- Session IDs: CSPRNG, ≥ 128 bits, rotated on login and re-auth, invalidated server-side on
  logout, and killed when the account is disabled (7.2.3, 7.2.4, 7.4.1, 7.4.2).
- Cookies: `Secure`, `__Host-` prefix, `HttpOnly` for tokens, `SameSite` per purpose (3.3.x).
- CSRF: state changes only via non-safe methods **plus** an anti-forgery token or a required
  custom header / preflight (3.5.1–3.5.3).

**Tokens and OAuth**
- JWT: verify the signature first; allowlist algorithms (never `none`, and prevent HS/RS
  confusion); load keys only from trusted config (ignore attacker `jku`/`jwk`/`x5u`); check
  `exp`/`nbf`, `aud`, and token type (9.1.x, 9.2.x).
- Self-contained tokens need a revocation story: a denylist, a per-user not-before time, or key
  rotation (7.4.1).
- OAuth: authorization code + **PKCE (S256)**; no implicit or password grants; exact-match
  redirect URIs; single-use, short-lived codes; refresh-token rotation or sender-constraining;
  BFF keeps tokens off the browser (10.4.x, 10.1.1). Identify users by `iss`+`sub` (10.3.3).

**Browser hardening (L1/L2)**
- HSTS ≥ 1 year on all responses (3.4.1); strict CORS origin allowlist (3.4.2); CSP with
  `object-src 'none'; base-uri 'none'` plus nonces/hashes or an allowlist (3.4.3); `frame-ancestors`
  rather than relying on `X-Frame-Options` (3.4.6); `nosniff` (3.4.4); a `Referrer-Policy` (3.4.5).
- Uploaded or API content can't render as HTML in the app's origin: `Content-Disposition:
  attachment`, CSP `sandbox`, or `Sec-Fetch-*` checks (3.2.1).

**Crypto and transport**
- AEAD only (AES-GCM, ChaCha20-Poly1305); no ECB or unauthenticated CBC; no MD5 or SHA-1 for
  security; ≥ 128-bit security strength (11.3.x, 11.4.x, 11.2.3). Use vetted libraries and design
  for algorithm agility (11.2.1, 11.2.2).
- Every unguessable value (tokens, reset links, API keys, OTPs) comes from a CSPRNG with ≥ 128
  bits. **UUIDs (even v4) are not secrets** (11.5.1).
- TLS 1.2+ everywhere, internal included, with no plaintext fallback, and **certificate
  validation never disabled** (12.1.1, 12.2.1, 12.3.1–12.3.2).

**Config, data, dependencies, errors**
- No secrets in source, images, or build artifacts; use a secrets manager (13.3.1). No `.git` in
  the deploy (13.4.1); debug, TRACE, directory listing, and API docs/actuator endpoints are off
  in prod (13.4.2–13.4.5).
- Sensitive data never in URLs or query strings (14.2.1); `Cache-Control: no-store` on sensitive
  responses (14.3.2); nothing sensitive in localStorage/sessionStorage/IndexedDB besides the
  session token (14.3.3); clear client state on logout (14.3.1).
- Patch SLAs exist and are met; SBOM maintained; prod ships no test/sample/dev code (15.1.1,
  15.2.1, 15.1.2, 15.2.3).
- Log authn successes and failures, authz failures, and bypass attempts with
  when/where/who/what in UTC; never log credentials or payment data; mask tokens; encode
  against log injection; ship logs off-box (16.2.x–16.4.x).
- Generic error messages to clients; fail closed; a last-resort handler (16.5.x).

## Frequently misapplied points

- **"Escaping makes SQL safe."** Not for identifiers. Allowlist table and column names (1.2.4 note).
- **"We validate input, so we don't need encoding."** Validation is defense in depth; encoding
  and parameterization are the fix (V1 objective, 2.2 intro).
- **"Hidden button = access control."** Authorization must happen server-side per request
  (8.3.1). Client-side checks are UX.
- **"UUIDs are unguessable IDs."** They don't meet 11.5.1. Fine as identifiers with real
  authorization checks; never as secrets or capability links.
- **"Strong passwords need symbols and 90-day rotation."** Both are prohibited by 6.2.5 and
  6.2.10. Length plus breach checks plus MFA is the model.
- **"`X-Frame-Options` handles clickjacking."** Obsolete per 3.4.6; use CSP `frame-ancestors`.
- **"HTTP→HTTPS redirect everywhere."** Only for browser-facing endpoints. APIs should fail
  instead of silently redirecting leaked plaintext requests (4.1.2).
- **"A JWT is encrypted."** A signed JWT payload is just Base64. Classify it as readable data
  (14.1.1).
- **"Our proxy sets `X-Forwarded-For`, so we trust it."** Only if the client can't inject it.
  Strip or overwrite at the edge (4.1.3, 15.3.4).
- **"The service account can read it, so the user can."** Decide on the *originating* user's
  permissions; propagate their token downstream (8.3.3, L3).

## Finding / requirement output format

When reporting a gap (review) or writing a requirement (design), use:

```
[v5.0.0-8.2.2 | L1 | FAIL] Object-level authorization missing on GET /api/invoices/{id}
Where:    src/invoices/controller.ts:42 (fetches by id, never checks invoice.accountId == session.accountId)
Impact:   Any authenticated user can read any invoice by iterating ids (IDOR/BOLA).
Fix:      Scope the query by the caller's account (findOne({id, accountId})) via the central policy layer; add a test that user B gets 404 for user A's invoice.
```

For a full assessment, produce a table with columns `ID | Level | Status (Pass/Fail/N/A/Not
verified) | Evidence / Notes`, grouped by chapter, preceded by scope, target level, exclusions
with rationale, and methods used.

## Beyond ASVS (Appendix D recommendations)

These aren't requirements, but they're recommended. Publish a `security.txt`; add a
password-strength meter; keep client-side validation too (it makes tampering detectable); keep
sensitive pages out of search engines; put GraphQL authorization in the business layer. Process
items (SSDLC, threat modeling per design change, security acceptance criteria in user stories,
drift detection, third-party hardening) belong to OWASP SAMM, not ASVS. Recommend them, but don't
score them as ASVS failures.

## Source and license

Distilled from *OWASP Application Security Verification Standard 5.0.0* (May 2025), © 2008–2025
The OWASP Foundation, licensed **CC BY-SA 4.0**
(<https://creativecommons.org/licenses/by-sa/4.0/>). The `references/` tables paraphrase and
condense the requirement text and Appendix C. They are adaptations shared under the same
license. For authoritative wording, use the official release at
<https://github.com/OWASP/ASVS> (CSV/JSON exports are available for tooling). Implementation
how-tos: OWASP Cheat Sheet Series. Test procedures: OWASP Web Security Testing Guide.
