---
name: public-key-cryptography-and-tls-handshakes
description: Explain or reason about how asymmetric/public-key cryptography (RSA, trapdoor functions, digital signatures) solves the "shared key problem" and how TLS/HTTPS uses it to establish a secure session. Use this when the user asks "how does HTTPS actually work", "what happens during a TLS handshake", "how does RSA encryption work", "why do we need both symmetric and asymmetric crypto", "what is a certificate authority / why do we trust certificates", "what's a man-in-the-middle attack and how does TLS prevent it", "what's the difference between encrypting with a public key vs a private key", or is debugging/reviewing certificate validation, mutual TLS, or a "why does my client reject this certificate" issue. Also trigger for questions about trapdoor/one-way functions, prime factorization hardness, or why RSA is combined with AES instead of used directly on bulk data.
---

# Public-Key Cryptography and TLS Handshakes

Symmetric ciphers like AES need both parties to already share a secret key — but two strangers
on the internet (your browser and a shop's server) have no prior shared secret and no secure
channel to exchange one over. This is the **shared key problem**, and public-key (asymmetric)
cryptography is the tool that solves it: an encryption key that can be handed to attackers
freely (the **public key**) and a separate decryption key that never leaves its owner (the
**private key**). TLS is the protocol that uses this to bootstrap a fast symmetric session
between two parties who've never met.

## The math: trapdoor functions, not "cleverness"

Public-key crypto needs a function that's a **one-way function** — easy to compute forward,
infeasible to invert — but with a **trapdoor**: infeasible to invert *unless* you know a secret
that makes the inverse easy. Squaring vs. square-rooting is a familiar one-way function but
has no trapdoor. RSA's trapdoor is prime factorization:

- Multiplying two large primes together is fast (`p x q = n`).
- Factoring `n` back into `p` and `q`, if you're only given `n`, requires checking candidate
  divisors and is infeasible for large enough primes (thousands of digits).
- But if you *already know* one of the primes, finding the other is trivial (just divide).

This asymmetry — hard in general, easy with a secret — is the entire security foundation of
RSA. Everything else (choosing keys, encrypting, decrypting) is arithmetic built on top of it.

## RSA in minimal working form

RSA key generation, encryption, and decryption, worked in code so the relationship between
public key, private key, and the modulus is concrete rather than abstract:

```python
from math import gcd

def generate_keypair(p: int, q: int):
    n = p * q                                  # the "prime-product" / RSA modulus — public
    totient = (p - 1) * (q - 1)                # only fast to compute if you know p and q
    e = 3                                      # public exponent: must be coprime with totient
    while gcd(e, totient) != 1:
        e += 2
    d = pow(e, -1, totient)                    # private exponent: modular inverse of e
    return (e, n), (d, n)                      # (public key), (private key)

def encrypt(message: int, public_key) -> int:
    e, n = public_key
    return pow(message, e, n)                  # message^e mod n

def decrypt(ciphertext: int, private_key) -> int:
    d, n = private_key
    return pow(ciphertext, d, n)                # ciphertext^d mod n

# Demo with small (INSECURE — illustration only) primes:
public_key, private_key = generate_keypair(97, 113)
ciphertext = encrypt(1482, public_key)          # anyone can do this — e, n are public
plaintext = decrypt(ciphertext, private_key)    # only the private-key holder can do this
assert plaintext == 1482
```

Real RSA uses primes hundreds of digits long specifically because factoring `n` must remain
infeasible; the toy primes above (97, 113) exist purely to make the arithmetic followable by
hand, not because small primes are ever acceptable in practice.

## Encryption vs. authentication: same math, keys swapped

RSA gives you two different guarantees depending on which key you encrypt with:

| Operation | Encrypt with | Decrypt with | Guarantee |
|---|---|---|---|
| Confidentiality | recipient's **public** key | recipient's **private** key | only the intended recipient can read it |
| Authentication (signing) | sender's **private** key | sender's **public** key | only the claimed sender could have produced it |

For confidentiality: anyone can encrypt a message to you because your public key is public,
and only you can decrypt it. For authentication: only you can produce a value that your
public key correctly "decrypts" into the expected result, because only you hold the private
key — this is a **digital signature**. Combine both (encrypt with the sender's private key,
then again with the recipient's public key) to get a message that's simultaneously
confidential and provably from the claimed sender. In practice, real signature schemes sign a
*hash* of the message rather than the message itself, for the same performance reasons
described next.

## Why RSA is never used directly on bulk data

RSA's core operation is modular exponentiation — computationally far more expensive than
AES's XOR/shift/table-lookup primitives. RSA can only encrypt data smaller than its modulus (a
few hundred bytes for practical key sizes), and even at that size it's orders of magnitude
slower than a block cipher per byte processed. **The universal pattern**: use RSA only to
securely transmit a short, randomly generated *symmetric* key, then switch to AES (or another
fast cipher) for the actual bulk data.

```python
# Hybrid encryption pattern — this is what every real-world "RSA-secured" transfer does.
def send_secure_document(document_bytes, recipient_rsa_public_key):
    aes_key = generate_random_aes_key()                          # fast, symmetric
    encrypted_doc = aes_encrypt(document_bytes, aes_key)          # bulk data -> AES
    encrypted_aes_key = rsa_encrypt(aes_key, recipient_rsa_public_key)  # tiny payload -> RSA
    return encrypted_aes_key, encrypted_doc
    # Recipient: rsa_decrypt(encrypted_aes_key, private_key) -> aes_key
    #            aes_decrypt(encrypted_doc, aes_key) -> document_bytes
```

This is not an implementation detail to skip past — it's the reason "asymmetric crypto" and
"symmetric crypto" aren't competing choices; production systems always use both together, RSA
(or Diffie-Hellman/ECDHE in modern TLS) for the key exchange and AES for the data.

## Certificates and the identity problem

Public-key crypto lets you encrypt to *a* public key, but doesn't by itself tell you *whose*
key it is. An attacker can hand you their own public key while claiming to be your bank — this
is the setup for a **man-in-the-middle (MITM) attack**: the attacker decrypts, reads, possibly
alters, then re-encrypts traffic to the real destination, invisible to both ends unless
identity is verified.

A **certificate authority (CA)** solves this by acting as a trusted third party that signs a
binding between a domain name and a public key (a **certificate**). Your browser ships with a
built-in list of trusted root CAs; when a server presents a certificate, your browser checks
that the certificate's signature validates against a CA it already trusts. Trust here is not
mathematical — it's a **social/administrative** assumption ("I trust that this CA verified the
domain owner before signing"), and it's the actual root of the entire chain. If a CA is
compromised or issues a certificate carelessly, every certificate it ever signed is suspect —
this is why CA compromise is one of the most severe possible failures in the web PKI system.

## The TLS handshake, step by step

The handshake authenticates the server (and optionally the client) and derives a shared
symmetric key, without ever transmitting that key itself:

1. **Client Hello** — client sends supported cipher suites and a random number (`client_random`).
2. **Server Hello + certificate** — server responds with its chosen cipher suite, its
   certificate (containing its public key, domain name, and CA signature), and its own random
   number (`server_random`).
3. **Certificate validation** — client checks the certificate's signature against a trusted CA
   and confirms the domain matches. If this fails, the connection should be aborted — silently
   proceeding here is exactly what defeats the whole point of the handshake.
4. **Key exchange** — client generates a random **premaster secret**, encrypts it with the
   server's public key (classic RSA key exchange) or, in modern TLS, derives it via an
   ephemeral Diffie-Hellman exchange (ECDHE) that additionally provides **forward secrecy**
   (a compromised long-term key can't retroactively decrypt past sessions, because the
   premaster secret was never transmitted at all, encrypted or otherwise).
5. **Master secret derivation** — both sides independently compute a **master secret** by
   hashing the premaster secret together with both random numbers. The master secret itself is
   never transmitted; both ends arrive at the same value only because they hold the same
   inputs.
6. **Session keys derived** — the master secret is split into a symmetric encryption key, a
   block-chaining initialization value, and a **MAC key** for per-message integrity checking
   (detecting tampering in transit, distinct from the identity authentication already done via
   the certificate).

```
Client                                              Server
  |--- ClientHello (cipher suites, client_random) -->|
  |<-- ServerHello (cipher suite, server_random,     |
  |     certificate w/ public key + CA signature) ---|
  |  [validate certificate against trusted CA store] |
  |--- premaster secret, encrypted w/ server pubkey ->|
  |  [both sides derive master secret independently] |
  |<==== symmetric session, AES + MAC per message ===>|
```

Once the handshake completes, all traffic uses fast symmetric encryption (AES) plus a MAC —
the expensive asymmetric operations happen exactly once per session, not per message.

## Common pitfalls

- **Treating "HTTPS" as a black box that "just makes things secure."** If you can't say what
  the certificate proves (identity of the domain, not trustworthiness of its content or
  operator) or what forward secrecy buys you, you can't reason about what a TLS failure
  actually means when debugging one.
- **Ignoring or suppressing certificate validation errors** ("just disable SSL verification to
  get past this error") — this removes the entire MITM defense and should never be a
  production fix, even temporarily; find and fix the actual cert/hostname/trust-store issue.
- **Assuming encryption implies authentication, or vice versa.** RSA encryption (public key)
  gives confidentiality; RSA signing (private key) gives authentication. TLS needs both:
  encryption for the data, the certificate chain for identity, and the MAC for per-message
  integrity — these are three separate guarantees, not one.
- **Using RSA directly on bulk data** instead of the hybrid RSA-for-key-exchange +
  AES-for-data pattern — slow, and typically not even supported by the RSA modulus size for
  anything beyond a small payload.
- **Not distinguishing static RSA key exchange from ephemeral (EC)DHE.** Static RSA key
  exchange means a compromised server private key can retroactively decrypt every past
  recorded session; ephemeral exchange (forward secrecy) means it can't. This matters
  concretely when choosing/auditing cipher suites.
- **Trusting a certificate because "the lock icon showed up"** without understanding that the
  lock only certifies the CA-verified identity of the domain, not that the domain's operator
  or content is trustworthy — phishing sites can and do have valid certificates for their own
  (differently spelled) domains.

## Checklist

- [ ] Can explain which key (public or private) is used for confidentiality vs. authentication
      in a given exchange, and why they're not interchangeable
- [ ] Confirmed bulk data uses a symmetric cipher, with RSA/ECDHE used only to establish that
      symmetric key
- [ ] Certificate validation failures are treated as hard stops, never bypassed in any
      environment that resembles production
- [ ] Understands the certificate proves domain identity via CA trust, not content
      trustworthiness
- [ ] Checked whether the deployment uses ephemeral key exchange (forward secrecy) vs. static
      RSA key exchange, and knows the risk difference
- [ ] Can walk through the handshake steps well enough to say which step a given TLS failure
      (cert error, cipher mismatch, handshake timeout) is actually failing at
