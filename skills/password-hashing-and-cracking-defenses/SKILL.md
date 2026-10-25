---
name: password-hashing-and-cracking-defenses
description: Design or review how passwords are hashed, salted, and stored for authentication, and reason about the attacks (dictionary attacks, rainbow tables/precomputed hash chains, brute force) that a scheme must resist. Use this when the user asks "how should I store user passwords", "what's the difference between hashing and encryption", "why do we salt passwords", "what is a rainbow table and how does salting defeat it", "how many hash iterations / what bcrypt/scrypt/argon2 cost factor should I use", is reviewing an authentication system or a database schema with a `password` or `password_hash` column, is implementing a login/signup flow, or is explaining why a leaked password database is or isn't catastrophic. Also trigger for "what makes a good hash function", collision attacks on file signatures/checksums, or comparing MD5/SHA-1/SHA-256 for non-password use cases like file integrity.
---

# Password Hashing and Cracking Defenses

A password authentication system's core problem: you must be able to verify a password
without storing it in a form that's useful to an attacker who steals your database. The
answer is a one-way transformation (hashing), but hashing alone is not enough — real systems
need salting and deliberately slow, iterated hashing to resist the attacks described below.
Treat "how do I store passwords" as a question with a well-established, non-negotiable answer;
inventing your own scheme here is one of the highest-blast-radius mistakes in software.

## Hashing vs. encryption: know which property you need

Encryption is reversible by design (with the key); hashing must be **one-way** — there should
be no feasible way to recover an input from its output. A good password hash function has
three properties:

- **Full bit usage**: output should use the entire range of the hash, not cluster in a subset.
- **Non-reversibility**: no shortcut to recover *a* password (not necessarily *the* password —
  collisions are inevitable since there are more possible passwords than hash values) that
  produces a given hash.
- **Avalanche**: a tiny input change (`dog` vs `Dog` vs `doge`) should produce a wildly
  different, uncorrelated output.

MD5 and SHA-1 meet the avalanche/full-use properties but have known collision weaknesses that
make them unsuitable where an attacker benefits from crafting *two* inputs with the same hash
(e.g., forging a file whose signature matches a trusted one). For **password storage**
specifically, the disqualifying problem with plain MD5/SHA-256 isn't collisions — it's that
they're too *fast*: designed for speed, which is exactly the wrong property when the threat
model is "attacker can compute billions of candidate hashes per second on stolen data."

## The three attacks a password store must resist

**1. Direct compromise (no hashing at all).** Storing passwords in plaintext, or reversibly
encrypted with a key stored alongside them, means anyone who reads the table can log in as
any user. This is not hypothetical — it keeps happening at large companies. If you ever see a
`password` column with human-readable values, that's an immediate, non-negotiable fix.

**2. Dictionary attacks.** An attacker without database access just tries common passwords
against the live login form. Defend with rate limiting / lockout after N failed attempts —
this is a runtime control, not a hashing-scheme control. An attacker *with* a stolen,
hashed table instead hashes an entire dictionary once and compares against every row —
cheap if the hash function is fast and unsalted, because one dictionary hash pass now cracks
every user who chose a common password.

**3. Precomputed tables (rainbow tables) and hash chains.** Building a full table of
`password -> hash` for a huge keyspace up front is infeasible to store directly (petabytes for
even a modest character set/length). **Hash chaining** compresses this: alternate a hash
function `H` with a *reduction function* `R` (hash -> plausible password) to build long chains,
storing only each chain's first and last entries. A target hash is searched for by re-running
`R`/`H` from the target until a stored chain-end matches, then replaying that chain from the
start to recover the actual password. This lets an attacker trade a large amount of upfront
computation for a small storage footprint that can be reused against *any* stolen table using
the same unsalted hash function — the entire point of the attack is amortizing cost across
many victims.

## Salting: making precomputed tables worthless

A **salt** is a random string generated per-account and combined with the password before
hashing (e.g., `hash(password + salt)`), then stored alongside the hash in plaintext — the
salt doesn't need to be secret, only unique per user.

```python
import hashlib, secrets

def hash_password(password: str) -> tuple[str, str]:
    salt = secrets.token_hex(16)                       # unique per user, stored alongside hash
    digest = hashlib.sha256((password + salt).encode()).hexdigest()
    return salt, digest

def verify_password(password: str, salt: str, expected_digest: str) -> bool:
    return hashlib.sha256((password + salt).encode()).hexdigest() == expected_digest
```

Why this defeats rainbow tables: a precomputed table is built for *one specific* hash
function with no salt. Once every account has a distinct salt, the attacker's precomputed
table is worthless — they'd need to build a full new table per salt, which for 100,000 users
with unique salts means 100,000 separate cracking efforts instead of one. This is why salting
is considered mandatory, not optional hardening.

**Common salting mistakes**: reusing one global salt for all users (defeats the purpose —
back to one precomputed table for everyone), using a predictable salt (e.g., the username),
or making the salt secret (unnecessary — its uniqueness, not its secrecy, is what matters, and
treating it as secret often leads to it being stored insecurely "for safety" while the actual
schema stays weak).

## Iterated hashing and modern KDFs: making each guess expensive

Salting stops precomputation but doesn't slow down an attacker who already has your table and
is trying candidate passwords against it. **Iterated hashing** — running the hash function
thousands of times per password — multiplies the attacker's per-guess cost by the same
factor, while adding a barely-perceptible delay to a legitimate login (a few hundred
milliseconds paid once at login time).

```python
def iterated_hash(password: str, salt: str, iterations: int = 200_000) -> str:
    digest = (password + salt).encode()
    for _ in range(iterations):
        digest = hashlib.sha256(digest).digest()
    return digest.hex()
```

In practice, don't hand-roll iteration like this — use a purpose-built password hashing
function that bundles salting, iteration, and (critically) memory-hardness:

- **bcrypt** — mature, widely available, tunable cost factor; the safe default when in doubt.
- **scrypt** / **Argon2** — additionally require large amounts of memory per guess, which
  specifically defeats attackers using GPUs/ASICs, whose advantage is massive *parallel*
  cheap compute but comparatively limited memory bandwidth per unit.

The cost-factor tuning problem is genuinely hard: pick a work factor that's a real burden for
an attacker running billions of parallel guesses, but still fast enough that a legitimate user
doesn't notice at login. There's no future-proof fixed answer — hardware gets faster, so cost
factors need periodic increases, which is why modern libraries store the cost factor alongside
the hash (so it can be upgraded incrementally as users log in, rather than requiring a
one-shot rehash of the entire user table).

## Common pitfalls

- **Storing passwords in plaintext or with reversible encryption.** If you can retrieve a
  user's original password from your database, your storage is wrong regardless of what else
  is done well.
- **Hashing without a salt.** Makes every account vulnerable to a single shared precomputed
  table/rainbow table attack instead of requiring per-account effort.
- **Using a fast, general-purpose hash (MD5, SHA-1, plain SHA-256) for password storage.**
  These are optimized for speed, which directly helps an attacker brute-forcing a stolen
  table. Use bcrypt/scrypt/Argon2, which are deliberately slow and (for scrypt/Argon2)
  memory-hard.
- **Confusing "hash for password storage" with "hash for file integrity/signatures."** File
  signature use cases (verifying a download matches what the publisher built) want a *fast*,
  collision-resistant hash (SHA-256), not a slow one — different threat model, different tool.
- **Relying on iteration count alone against modern hardware.** Straight iterated SHA-256 is
  vulnerable to cheap GPU/ASIC parallelism because each guess needs almost no memory; prefer
  memory-hard KDFs (scrypt/Argon2) specifically to blunt that advantage.
- **No rate limiting on login attempts.** Even a perfectly salted, slow hash doesn't stop an
  online dictionary attack against the live login form if there's no lockout/backoff — hashing
  strength and rate limiting are separate, both-required defenses.
- **Treating a hashed table as fully safe to leak.** Hashing raises the cost of attack; it
  doesn't make a leak a non-event. Weak/reused passwords are still crackable given enough
  time, which is why breach response includes forced password resets, not just "don't worry,
  it was hashed."

## Checklist

- [ ] Passwords are never stored in plaintext or reversibly encrypted
- [ ] Every password hash uses a unique, randomly generated, non-secret salt
- [ ] The hash function is a purpose-built, deliberately slow/memory-hard KDF (bcrypt, scrypt,
      or Argon2) — not a fast general-purpose hash
- [ ] The work factor/cost parameter is stored alongside the hash so it can be upgraded later
      without breaking existing users
- [ ] Login endpoints have rate limiting / lockout independent of hash strength
- [ ] File-integrity/signature use cases use a fast collision-resistant hash (SHA-256), not a
      slow KDF — the two problems require opposite properties
