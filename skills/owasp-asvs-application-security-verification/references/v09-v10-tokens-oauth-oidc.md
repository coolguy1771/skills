# ASVS 5.0.0 — V9–V10: Self-contained Tokens, OAuth and OIDC

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`.

## V9 Self-contained Tokens (JWT, SAML assertions, etc.)

A self-contained token carries claims the receiver trusts for security decisions, unlike a
reference token that is only an ID to look up. All of its safety rests on integrity and
context validation.

### V9.1 Token Source and Integrity
| ID | L | Requirement |
|---|---|---|
| 9.1.1 | 1 | Validate the signature or MAC **before** trusting any contents. |
| 9.1.2 | 1 | Only allowlisted algorithms per context. **Never `none`.** Prefer only symmetric or only asymmetric; if both are needed, add controls against key confusion (e.g., HS256 verified with an RSA public key). |
| 9.1.3 | 1 | Verification keys come from trusted, pre-configured sources for that issuer. JWS `jku`, `x5u`, and `jwk` headers are validated against an allowlist; attackers can't supply their own keys. |

### V9.2 Token Content
| ID | L | Requirement |
|---|---|---|
| 9.2.1 | 1 | If a validity window exists, enforce it (JWT `nbf` and `exp`). |
| 9.2.2 | 2 | Check token **type/purpose**: only access tokens for authorization, only ID tokens for proving authentication. |
| 9.2.3 | 2 | Accept only tokens whose **audience** is this service (`aud` checked against a service-side allowlist). |
| 9.2.4 | 2 | An issuer that signs for multiple audiences with one key must include a unique audience restriction. Dynamically provisioned audiences are validated to prevent audience impersonation. |

## V10 OAuth and OIDC

Roles: **client** (confidential = can keep a secret, e.g., a backend/BFF; public = can't, e.g.,
an SPA or native app), **resource server (RS)** = the API, **authorization server (AS)** issues
tokens, **resource owner** = the user. OIDC adds the **relying party (RP)** (a client) and the
**OpenID Provider (OP)** (an AS that authenticates users). Token audiences: access tokens → RS
only; refresh tokens → the issuing AS only; ID tokens → the client that started the flow only.

Use a well-known, industry-standard authorization server with its recommended security
configuration rather than building one. Align with the current RFCs: RFC 9700 (OAuth 2.0 Security
BCP), OAuth 2.1 draft, OAuth for Browser-Based Apps draft, FAPI 2.0.

### V10.1 Generic
| ID | L | Requirement |
|---|---|---|
| 10.1.1 | 2 | Tokens go only to components that need them. With a BFF for a browser SPA, access and refresh tokens stay on the backend. |
| 10.1.2 | 2 | The client accepts AS values (code, ID token) only from a flow started by the **same user-agent session and transaction**. PKCE `code_verifier`, `state`, and `nonce` are unguessable, per-transaction, and bound to client and session. |

### V10.2 OAuth Client
| ID | L | Requirement |
|---|---|---|
| 10.2.1 | 2 | Code flow is CSRF-protected via **PKCE** or by checking `state`. |
| 10.2.2 | 2 | A client talking to more than one AS defends against **mix-up** attacks (e.g., require and validate `iss` in the authorization and token responses). |
| 10.2.3 | 3 | Request only the scopes/authorization parameters actually needed. |

### V10.3 OAuth Resource Server (after token validation, before authorization)
| ID | L | Requirement |
|---|---|---|
| 10.3.1 | 2 | Accept only access tokens whose audience is this RS (`aud` claim or introspection). |
| 10.3.2 | 2 | Authorization decisions use the delegated-authorization claims; `sub`, `scope`, and `authorization_details` must be part of the decision when present. |
| 10.3.3 | 2 | Identify a unique user from non-reassignable claims, typically **`iss` + `sub`**. |
| 10.3.4 | 2 | If specific auth strength, method, or recency is required, verify it (`acr`, `amr`, `auth_time`). |
| 10.3.5 | 3 | Require **sender-constrained** access tokens (mTLS-bound or DPoP) against theft and replay. |

### V10.4 OAuth Authorization Server
| ID | L | Requirement |
|---|---|---|
| 10.4.1 | 1 | Redirect URIs are validated against a per-client pre-registered allowlist by **exact string comparison**. |
| 10.4.2 | 1 | An authorization code is single-use. On a second use, reject it **and revoke** all tokens issued from it. |
| 10.4.3 | 1 | Authorization codes are short-lived: ≤ 10 min (L1/L2), ≤ 1 min (L3). |
| 10.4.4 | 1 | Each client may use only the grants it needs. **Implicit (`token`) and ROPC (`password`) must not be used.** |
| 10.4.5 | 1 | Refresh-token replay is mitigated for public clients, preferably with sender-constrained tokens (DPoP/mTLS). L1/L2 may use rotation: invalidate each refresh token on use, and revoke the whole grant if a used one is replayed. |
| 10.4.6 | 2 | Code grant requires **PKCE**: a valid `code_challenge`, **`plain` rejected** (S256), and `code_verifier` checked at the token endpoint. |
| 10.4.7 | 2 | Unauthenticated dynamic client registration validates client metadata (URIs), ensures consent, and warns users about untrusted clients. |
| 10.4.8 | 2 | Refresh tokens have an **absolute** expiry, even with sliding expiration. |
| 10.4.9 | 2 | Users can revoke refresh tokens and reference access tokens in the AS UI. |
| 10.4.10 | 2 | Confidential clients authenticate on back-channel calls (token, PAR, revocation). |
| 10.4.11 | 2 | The AS assigns each client only the scopes it requires. |
| 10.4.12 | 3 | Only the `response_mode` each client needs is allowed (validated, or enforced via PAR/JAR). |
| 10.4.13 | 3 | Code grant is always used with **PAR** (pushed authorization requests). |
| 10.4.14 | 3 | Only sender-constrained (PoP) access tokens are issued: mTLS-bound or DPoP. |
| 10.4.15 | 3 | For server-side clients, `authorization_details` is guaranteed to come untampered from the client backend (PAR or JAR). |
| 10.4.16 | 3 | Clients are confidential and use strong public-key, replay-resistant client authentication: `tls_client_auth`, `self_signed_tls_client_auth` (per RFC 8705 §2.2), or `private_key_jwt`. |

### V10.5 OIDC Client (Relying Party); V10.2 also applies
| ID | L | Requirement |
|---|---|---|
| 10.5.1 | 2 | Mitigate ID token replay: the `nonce` claim matches the nonce sent in the authentication request. |
| 10.5.2 | 2 | Identify the user by a non-reassignable claim, usually `sub` (unique within the IdP). |
| 10.5.3 | 2 | Reject AS metadata whose issuer URL doesn't **exactly** match the pre-configured issuer (impersonation via metadata). |
| 10.5.4 | 2 | ID token `aud` equals this client's `client_id`. |
| 10.5.5 | 2 | Back-channel logout: the token has `typ` `logout+jwt`, an `events` claim with the correct member, and **no** `nonce` claim. Short expiry (~2 min) is recommended. Prevents forced-logout DoS and cross-JWT confusion. |

### V10.6 OpenID Provider; V10.4 also applies
| ID | L | Requirement |
|---|---|---|
| 10.6.1 | 2 | Only `code`, `ciba`, `id_token`, or `id_token code` responses are allowed. `code` is preferred over the hybrid flow; `token` (any implicit flow) must not be used. |
| 10.6.2 | 2 | Forced-logout DoS is mitigated: explicit user confirmation, or validation of RP-initiated logout parameters such as `id_token_hint`. |

### V10.7 Consent Management (AS)
| ID | L | Requirement |
|---|---|---|
| 10.7.1 | 2 | The user consents to each authorization request. If client identity can't be assured, always prompt explicitly. |
| 10.7.2 | 2 | Consent prompts clearly show what's granted: scopes, RS, RAR `authorization_details`, client identity, and lifetime. |
| 10.7.3 | 2 | Users can review, modify, and revoke granted consents. |
