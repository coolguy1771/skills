# ASVS 5.0.0 — V11–V12: Cryptography and Secure Communication (+ Appendix C)

Condensed from OWASP ASVS 5.0.0 (CC BY-SA 4.0). `L` = the lowest level at which the requirement
applies. Cite as `v5.0.0-<id>`. The algorithm tables at the end are ASVS Appendix C. They define
"approved" for this chapter and may be overridden by regulation or new cryptanalysis.

## V11 Cryptography

### V11.1 Cryptographic Inventory and Documentation
| ID | L | Requirement |
|---|---|---|
| 11.1.1 | 2 | A documented key management policy and key lifecycle following a standard like NIST SP 800-57. Keys aren't overshared: ≤ 2 entities for shared secrets, 1 for private keys. |
| 11.1.2 | 2 | A maintained crypto inventory of all keys, algorithms, and certificates, recording where each key may and may not be used and what data it may protect. |
| 11.1.3 | 3 | Discovery mechanisms (static/dynamic scanning, CBOM tooling) find every use of encryption, hashing, and signing. |
| 11.1.4 | 3 | The inventory includes a migration plan to new standards, especially **post-quantum**. |

### V11.2 Secure Implementation
| ID | L | Requirement |
|---|---|---|
| 11.2.1 | 2 | Use industry-validated crypto implementations (libraries, hardware). No home-grown crypto. |
| 11.2.2 | 2 | **Crypto agility:** RNG, AEAD, MAC, hash, key lengths, rounds, ciphers, and modes can be reconfigured or swapped. Keys and passwords can be replaced and data re-encrypted (PQC readiness). |
| 11.2.3 | 2 | All primitives provide ≥ **128 bits of security** (e.g., ECC 256-bit, RSA 3072-bit). |
| 11.2.4 | 3 | Crypto operations are constant-time, with no short-circuit comparisons, calculations, or returns. |
| 11.2.5 | 3 | Crypto modules fail securely; errors enable no oracles (e.g., padding oracle). |

### V11.3 Encryption Algorithms
| ID | L | Requirement |
|---|---|---|
| 11.3.1 | 1 | No insecure block modes (**ECB**) or weak padding (**PKCS#1 v1.5**). |
| 11.3.2 | 1 | Only approved ciphers and modes, e.g., **AES-GCM**. |
| 11.3.3 | 2 | Encrypted data is protected against modification, preferably with AEAD, or with an approved cipher plus an approved MAC. |
| 11.3.4 | 3 | Nonces/IVs are never reused for the same key; generation suits the algorithm. |
| 11.3.5 | 3 | Any cipher+MAC combination uses **encrypt-then-MAC**. |

### V11.4 Hashing and Hash-based Functions
| ID | L | Requirement |
|---|---|---|
| 11.4.1 | 1 | Only approved hashes for crypto uses (signatures, HMAC, KDF, RBG). **MD5 is disallowed for any cryptographic purpose.** |
| 11.4.2 | 2 | Passwords are stored with an approved, computationally intensive password-hashing KDF, with parameters per current guidance (see table below). |
| 11.4.3 | 2 | Hashes for signatures, data authentication, or integrity are collision-resistant: ≥ **256-bit** output when collision resistance is needed, ≥ 128-bit when only second-preimage resistance is. |
| 11.4.4 | 2 | Keys derived from passwords use an approved KDF with key-stretching parameters. |

### V11.5 Random Values
| ID | L | Requirement |
|---|---|---|
| 11.5.1 | 2 | Every value meant to be non-guessable comes from a **CSPRNG** with ≥ **128 bits** of entropy. **UUIDs do not qualify.** |
| 11.5.2 | 3 | The RNG stays secure under heavy demand. |

### V11.6 Public Key Cryptography (non-TLS uses)
| ID | L | Requirement |
|---|---|---|
| 11.6.1 | 2 | Approved algorithms and modes for key generation, seeding, and signatures. Keygen must not produce weak keys (e.g., Fermat-factorable RSA). |
| 11.6.2 | 3 | Approved key exchange (e.g., DH/ECDH) with secure parameters (prevents MITM and breaks). |

### V11.7 In-Use Data Cryptography
| ID | L | Requirement |
|---|---|---|
| 11.7.1 | 3 | Full memory encryption protects sensitive data in use. |
| 11.7.2 | 3 | Data minimization during processing; data is encrypted again as soon as possible after use. |

## V12 Secure Communication

Use separate TLS certificates per environment where possible. A wildcard cert shared across
prod/staging/dev widens the blast radius of a compromise.

### V12.1 General TLS
| ID | L | Requirement |
|---|---|---|
| 12.1.1 | 1 | Only current TLS versions (**TLS 1.2 and 1.3**), latest preferred. |
| 12.1.2 | 2 | Only recommended cipher suites, strongest preferred. **L3:** forward-secret suites only. |
| 12.1.3 | 2 | mTLS client certificates are validated as trusted **before** their identity is used for authn/authz. |
| 12.1.4 | 3 | Certificate revocation checking is configured (e.g., OCSP stapling). |
| 12.1.5 | 3 | Encrypted Client Hello (ECH) hides SNI and other metadata. |

### V12.2 HTTPS to External-Facing Services
| ID | L | Requirement |
|---|---|---|
| 12.2.1 | 1 | TLS for all client ↔ external HTTP services, with **no fallback** to plaintext. |
| 12.2.2 | 1 | External services use publicly trusted certificates. |

### V12.3 Service-to-Service Communication
| ID | L | Requirement |
|---|---|---|
| 12.3.1 | 2 | Encrypted protocols (TLS) for **all** inbound and outbound connections: monitoring, management, SSH, middleware, databases, mainframes, partners, external APIs. No fallback. |
| 12.3.2 | 2 | TLS clients validate server certificates (no `verify=False` / `InsecureSkipVerify`). |
| 12.3.3 | 2 | TLS (or equivalent) between internal HTTP services, no fallback. |
| 12.3.4 | 2 | Internal TLS uses trusted certs. With internal CAs or self-signed certs, consumers trust **only** those specific CAs/certs. |
| 12.3.5 | 3 | Strong mutual authentication between internal services (mTLS / replay-resistant PKI); consider a service mesh. |

---

## Appendix C — Approved Algorithms Cheat Sheet

Status: **A** = approved, **L** = legacy (compatibility only; replace ASAP), **D** = disallowed.

### Equivalent strengths (NIST SP 800-57)
| Security bits | Symmetric | Finite field (DSA/DH) | RSA | ECC |
|---|---|---|---|---|
| ≤ 80 | 2TDEA | L=1024, N=160 | 1024 | 160–223 |
| 112 | 3TDEA | L=2048, N=224 | 2048 | 224–255 |
| **128** | AES-128 | L=3072, N=256 | **3072** | **256–383** |
| 192 | AES-192 | L=7680, N=384 | 7680 | 384–511 |
| 256 | AES-256 | L=15360, N=512 | 15360 | 512+ |

(Assumes no cryptographically relevant quantum computer; the asymmetric columns break if one exists.)

### Random number generation (all A)
`/dev/urandom`, `/dev/random` (Linux 4.8+, ChaCha20-based), `getentropy()` (OpenBSD, glibc 2.25+,
macOS 10.12+), AES-CTR-DRBG / HMAC-DRBG / Hash-DRBG (NIST SP 800-90A, e.g., Windows
`BCryptGenRandom`). In practice, use the language's CSPRNG API (`secrets`, `crypto.randomBytes`,
`crypto/rand`, `SecureRandom`), which wraps these.

### Symmetric ciphers (in preference order)
A: AES-256, Salsa20, XChaCha20, XSalsa20, ChaCha20, AES-192 · **L:** AES-128 ·
**D:** 2TDEA, 3DES/TDEA, IDEA, RC4/ARC4, Blowfish, DES.

### AES modes
A: **GCM**, CCM (both authenticated) · **L:** CBC (only with a MAC, encrypt-then-MAC, and
constant-time padding checks) · **D:** CCM-8 (64-bit tag), ECB, CFB, OFB, CTR (unauthenticated).
Every encrypted message must be authenticated; disk encryption (XTS etc.) is out of scope.

### Key wrapping
AES-256 with **KW** or **KWP** (NIST SP 800-38F). AES-128/192 only with a documented reason in the
crypto inventory.

### Authenticated encryption
A: AES-GCM, AES-CCM, **ChaCha20-Poly1305**, AEGIS-256/128/128L, Encrypt-then-MAC ·
**L:** MAC-then-encrypt (TLS 1.2 legacy suites).

### General-purpose hashes
A: SHA3-512/384/256, SHA-512, SHA-384, SHA-512/256, SHA-256, SHAKE256, BLAKE2s, BLAKE2b, BLAKE3 ·
**L** (not for HMAC/KDF/RBG/signatures): SHA-224, SHA-512/224, SHA3-224, **SHA-1** ·
**D:** CRC (any length), MD4, **MD5**. Outputs under 254 bits must not be used where collision
resistance is needed.

### Password hashing / password-based KDFs (minimum parameters)
| Function | Required parameters | Status |
|---|---|---|
| **argon2id** (RFC 9106) | t=1: m ≥ 46 MiB, p=1 · t=2: m ≥ 19 MiB, p=1 · t ≥ 3: m ≥ 12 MiB, p=1 | A |
| scrypt (RFC 7914) | p=1: N ≥ 2^17 (128 MiB), r=8 · p=2: N ≥ 2^16 · p ≥ 3: N ≥ 2^15 | A |
| bcrypt | cost ≥ 10 (password storage only; note 72-byte input limit) | A |
| PBKDF2-HMAC-SHA-512 | ≥ 210,000 iterations | A |
| PBKDF2-HMAC-SHA-256 | ≥ 600,000 iterations | A |
| PBKDF2-HMAC-SHA-1 | ≥ 1,300,000 iterations | L |

(For password-based **key derivation**, the appendix lists argon2id t=1/t=2, scrypt, and PBKDF2;
bcrypt is for storage only.)

### General KDFs
A: **HKDF** · L: TLS 1.2 PRF · D: MD5- or SHA-1-based KDFs.

### Key exchange (≥ 112-bit strength; NIST SP 800-56A/B, 800-77)
A: FFDH (L ≥ 3072, N ≥ 256), ECDH (f ≥ 256) · **D:** RSA PKCS#1 v1.5 key transport, IKEv1.
Approved groups: P-224/256/384/521, K-/B-233/283/409/571, **Curve25519**, Curve448,
MODP-2048…8192, ffdhe2048…8192.

### MACs
A: HMAC-SHA-256/384/512, KMAC128/256, BLAKE3 keyed, AES-CMAC, AES-GMAC, Poly1305-AES ·
**L:** HMAC-SHA-1 · **D:** HMAC-MD5.

### Signatures
A: **EdDSA (Ed25519, Ed448)**, XEdDSA, ECDSA (P-256/384/521), RSA-PSS ·
**D:** RSA PKCS#1 v1.5 signatures, DSA (any size).

### Post-quantum
Follow FIPS 203 (ML-KEM), 204 (ML-DSA), 205 (SLH-DSA). The hybrid `X25519MLKEM768` TLS key
agreement is shipping in major browsers and may be used where industry- or government-approved
libraries support it.
