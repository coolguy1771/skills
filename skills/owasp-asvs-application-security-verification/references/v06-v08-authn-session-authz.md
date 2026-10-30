# ASVS 5.0.0 — V6–V8: Authentication, Session Management, Authorization

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`.

**Level stance on authentication:** L1 leans on password hygiene. **L2 requires MFA.** **L3
requires hardware-based, phishing-resistant authentication** (device-bound passkeys, FIDO keys,
NIST AAL3 or equivalent). Relaxing MFA requires a documented rationale plus compensating controls.
Email is not an acceptable authentication mechanism (per NIST SP 800-63).

## V6 Authentication

### V6.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 6.1.1 | 1 | Document how rate limiting, anti-automation, and adaptive response defend against credential stuffing and brute force, how they're configured, and how they avoid **malicious account lockout**. |
| 6.1.2 | 2 | Document a list of context-specific words banned from passwords (org/product names, system IDs, codenames, departments, roles, permutations). |
| 6.1.3 | 2 | Document every authentication pathway, with the controls and strength enforced consistently across them. |

### V6.2 Password Security
| ID | L | Requirement |
|---|---|---|
| 6.2.1 | 1 | Minimum length 8; **15 strongly recommended**. |
| 6.2.2 | 1 | Users can change their password. |
| 6.2.3 | 1 | Password change requires the current **and** new password. |
| 6.2.4 | 1 | On registration/change, check against at least the top 3000 passwords that satisfy the policy. |
| 6.2.5 | 1 | **No composition rules.** No required upper/lower/digit/special counts; any characters allowed. |
| 6.2.6 | 1 | `type=password` masking; a temporary reveal (whole password or last char) is allowed. |
| 6.2.7 | 1 | Paste, browser password helpers, and password managers are permitted. |
| 6.2.8 | 1 | Verify the password exactly as received: no truncation, no case folding. |
| 6.2.9 | 2 | Passwords of at least 64 characters are permitted. |
| 6.2.10 | 2 | **No periodic rotation.** A password stays valid until it's known to be compromised or the user changes it. |
| 6.2.11 | 2 | Enforce the documented context-specific word list. |
| 6.2.12 | 2 | On registration/change, check against a breached-password set. |

### V6.3 General Authentication Security
| ID | L | Requirement |
|---|---|---|
| 6.3.1 | 1 | Implement the documented credential-stuffing and brute-force controls. |
| 6.3.2 | 1 | No default accounts (`root`, `admin`, `sa`), or they're disabled. |
| 6.3.3 | 2 | MFA (or a combination of single-factor mechanisms) is required for access. **L3:** one factor is hardware-based, phishing-resistant, and requires a user-initiated action (FIDO key button press, phone). Relaxations need documented rationale plus compensating controls. |
| 6.3.4 | 2 | No undocumented authentication pathways; strength enforced consistently across all of them. |
| 6.3.5 | 3 | Notify users of suspicious attempts: unusual location or client, partial success (one factor), attempt after long inactivity, success after several failures. |
| 6.3.6 | 3 | Email is not used as a single-factor or multi-factor mechanism. |
| 6.3.7 | 3 | Notify users after authentication details change (credential reset, username/email change). |
| 6.3.8 | 3 | **No user enumeration** via error messages, HTTP status codes, or response timing. This also covers registration and forgot-password. |

### V6.4 Factor Lifecycle and Recovery
| ID | L | Requirement |
|---|---|---|
| 6.4.1 | 1 | System-generated initial passwords and activation codes are CSPRNG-generated, follow the policy, expire quickly or on first use, and can't become the long-term password. |
| 6.4.2 | 1 | No password hints or knowledge-based "secret questions". |
| 6.4.3 | 2 | Forgotten-password reset is secure and **doesn't bypass enabled MFA**. |
| 6.4.4 | 2 | Replacing a lost MFA factor requires identity proofing at the same level as enrollment. |
| 6.4.5 | 3 | Renewal instructions for expiring authenticators are sent early enough, with reminders. |
| 6.4.6 | 3 | Admins can **initiate** a reset but can't set or choose the user's password. |

### V6.5 General MFA Requirements (lookup secrets, TOTP, out-of-band)
| ID | L | Requirement |
|---|---|---|
| 6.5.1 | 2 | Lookup secrets, OOB requests/codes, and TOTPs are usable **once**. |
| 6.5.2 | 2 | Stored lookup secrets with < 112 bits of entropy are hashed with an approved password-hashing function and a 32-bit random salt. ≥ 112 bits may use a standard hash. |
| 6.5.3 | 2 | Lookup secrets, OOB codes, and TOTP seeds come from a CSPRNG. |
| 6.5.4 | 2 | Lookup secrets and OOB codes have ≥ 20 bits of entropy (6 random digits or 4 random alphanumerics). |
| 6.5.5 | 2 | Defined lifetimes: OOB ≤ **10 minutes**, TOTP ≤ **30 seconds**. |
| 6.5.6 | 3 | Any factor, including physical devices, can be revoked on theft or loss. |
| 6.5.7 | 3 | Biometrics are only a secondary factor, combined with something you have or know. |
| 6.5.8 | 3 | TOTP is checked against a trusted server time source, never client-supplied time. |

### V6.6 Out-of-Band Mechanisms
Email and VoIP are not permitted. SMS/PSTN is "restricted" per NIST. Be aware of push bombing.

| ID | L | Requirement |
|---|---|---|
| 6.6.1 | 2 | SMS/phone OTP only to a previously validated number, with a stronger alternative (e.g., TOTP) offered and risks explained. **L3:** no SMS or phone. |
| 6.6.2 | 2 | OOB requests/codes/tokens are bound to the original authentication request; not reusable for earlier or later ones. |
| 6.6.3 | 2 | Code-based OOB is rate-limited against brute force; consider codes with ≥ 64 bits of entropy. |
| 6.6.4 | 3 | Push-based MFA is rate-limited against push bombing; number matching helps. |

### V6.7 Cryptographic Authenticators (smart cards, FIDO)
| ID | L | Requirement |
|---|---|---|
| 6.7.1 | 3 | Certificates used to verify cryptographic assertions are stored tamper-proof. |
| 6.7.2 | 3 | Challenge nonce ≥ 64 bits, statistically unique over the device lifetime. |

### V6.8 Authentication with an Identity Provider
| ID | L | Requirement |
|---|---|---|
| 6.8.1 | 2 | With multiple IdPs, identify users by **(IdP ID, user ID in that IdP)** so one IdP can't spoof another's users. |
| 6.8.2 | 2 | Always validate the presence and integrity of signatures on assertions (JWT, SAML). Reject unsigned or invalid ones. |
| 6.8.3 | 2 | SAML assertions are processed once within their validity period (replay). |
| 6.8.4 | 2 | If specific strength, method, or recency is required, verify it from IdP data (OIDC `acr`, `amr`, `auth_time`). If the IdP doesn't provide it, a documented fallback assumes the weakest mechanism (single-factor password). |

## V7 Session Management

Cookie specifics live in V3.3. Self-contained token (JWT) specifics live in V9.

### V7.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 7.1.1 | 2 | Document the inactivity timeout and absolute max session lifetime, justified alongside other controls, including justification for any deviation from NIST SP 800-63B re-authentication. |
| 7.1.2 | 2 | Document the number of concurrent sessions allowed per account and what happens at the limit. |
| 7.1.3 | 2 | Document every system creating/managing sessions in a federated (SSO) ecosystem, and how lifetimes, termination, and re-auth are coordinated. |

### V7.2 Fundamental Session Security
| ID | L | Requirement |
|---|---|---|
| 7.2.1 | 1 | All session token verification happens in a trusted backend service. |
| 7.2.2 | 1 | Session tokens (self-contained or reference) are dynamically generated, never static API secrets/keys. |
| 7.2.3 | 1 | Reference tokens are unique, CSPRNG-generated, with ≥ **128 bits** of entropy. |
| 7.2.4 | 1 | Issue a **new** session token on authentication and re-authentication, and terminate the old one (session fixation). |

### V7.3 Session Timeout
| ID | L | Requirement |
|---|---|---|
| 7.3.1 | 2 | Inactivity timeout forces re-authentication per the documented decisions. |
| 7.3.2 | 2 | Absolute max lifetime forces re-authentication per the documented decisions. |

### V7.4 Session Termination
| ID | L | Requirement |
|---|---|---|
| 7.4.1 | 1 | After logout or expiry, the session is unusable. Stateful sessions: invalidate in the backend. Self-contained tokens: a revocation list, a per-user "not-before" timestamp, or per-user signing key rotation. |
| 7.4.2 | 1 | All sessions end when an account is disabled or deleted. |
| 7.4.3 | 2 | After any authentication factor change (password change/reset, MFA update), the user is offered termination of all other sessions. |
| 7.4.4 | 2 | Every authenticated page has easy, visible logout. |
| 7.4.5 | 2 | Admins can terminate sessions for one user or for all users. |

### V7.5 Defenses Against Session Abuse
| ID | L | Requirement |
|---|---|---|
| 7.5.1 | 2 | **Full re-authentication** before changing email, phone, MFA config, or other recovery-relevant attributes. |
| 7.5.2 | 2 | Users can view their active sessions and terminate any or all of them (after re-authenticating with ≥ 1 factor). |
| 7.5.3 | 3 | Step-up authentication (≥ 1 factor or secondary verification) before highly sensitive operations. |

### V7.6 Federated Re-authentication (RP/IdP code)
| ID | L | Requirement |
|---|---|---|
| 7.6.1 | 2 | RP/IdP session lifetime and termination behave as documented, forcing re-auth when needed (e.g., max time between IdP authentications). |
| 7.6.2 | 2 | Creating a session requires user consent or explicit action (no silent session creation). |

## V8 Authorization

ASVS recommends (Appendix D) one centralized access control mechanism that all requests pass
through, and attribute/feature-based checks, with permissions still allocated via roles.

### V8.1 Documentation
| ID | L | Requirement |
|---|---|---|
| 8.1.1 | 1 | Document rules for **function-level** and **data-specific** access based on consumer permissions and resource attributes. |
| 8.1.2 | 2 | Document **field-level** read/write rules, which may depend on other attributes (state/status). |
| 8.1.3 | 3 | Document the environmental/contextual attributes (time, location, IP, device) used in security decisions. |
| 8.1.4 | 3 | Document how contextual factors drive decisions: attributes, risk thresholds, and actions (allow, challenge, deny, step-up). |

### V8.2 General Authorization Design
| ID | L | Requirement |
|---|---|---|
| 8.2.1 | 1 | Function-level access is restricted to consumers with explicit permissions. |
| 8.2.2 | 1 | Data-specific access is restricted to consumers with explicit permission for that item (**IDOR / BOLA**). |
| 8.2.3 | 2 | Field-level access is restricted to explicit permissions (**BOPLA**: over-exposure and mass-assignment of properties). |
| 8.2.4 | 3 | Adaptive controls based on context are applied at session start **and** during the session, as documented. |

### V8.3 Operation-Level Authorization
| ID | L | Requirement |
|---|---|---|
| 8.3.1 | 1 | Authorization is enforced at a **trusted service layer**, never by controls the client can manipulate (client-side JS, hidden fields). |
| 8.3.2 | 3 | Changes to authorization-relevant values apply immediately. Where impossible (data baked into self-contained tokens), alert on and revert actions taken after access was lost. |
| 8.3.3 | 3 | Access decisions use the **originating subject's** permissions, not an intermediary's. Propagate the user's token downstream rather than swapping in a service token. |

### V8.4 Other Considerations
| ID | L | Requirement |
|---|---|---|
| 8.4.1 | 2 | Multi-tenant apps enforce cross-tenant controls; operations never affect tenants the consumer can't access. |
| 8.4.2 | 3 | Admin interfaces use layered security (continuous identity verification, device posture, contextual risk). Network location or "trusted endpoints" are never the sole factor. |
