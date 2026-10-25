---
name: modular-arithmetic-number-theory-for-crypto-and-checksums
description: Implement and debug the number-theory primitives behind checksums, hash design, and crypto math — modular arithmetic, the extended Euclidean algorithm and modular inverse, fast modular exponentiation, primality testing, and linear Diophantine equations. Use this whenever the user wants to implement a check-digit algorithm (Luhn, ISBN), asks why a hash table should use a prime size, needs to compute a modular inverse or solve ax ≡ b (mod m), is implementing RSA-style math or asks "how does modular exponentiation work," wants to check primality (trial division vs Miller-Rabin) for numbers of a given size, or is solving an integer equation like ax + by = c. Distinct from public-key-cryptography-and-tls-handshakes/symmetric-encryption-and-aes-internals (those cover protocols and cipher internals) — this is the underlying integer arithmetic those protocols, plus everyday checksums and hash functions, are built from.
---

# Modular Arithmetic and Number Theory for Crypto and Checksums

Number theory shows up in ordinary application code far more often than "cryptography" alone
suggests: hash table sizing, credit-card and ISBN check digits, cyclic buffer indexing, and RNG
period lengths are all modular-arithmetic problems in disguise. This skill covers the primitives
— modular arithmetic, GCD/extended Euclid, modular exponentiation, primality testing, and linear
Diophantine equations — and the everyday and crypto-math problems they solve.

## GCD, LCM, and the Euclidean algorithm

```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

def lcm(a, b):
    return a * b // gcd(a, b)
```

`math.gcd` and `math.lcm` exist in the standard library — implement it by hand only to understand
the mechanism (each step replaces `(a, b)` with `(b, a mod b)`, which provably terminates and
preserves the GCD). Direct uses: reducing a fraction to lowest terms, finding when two periodic
events next align (`lcm` of their periods), and as the foundation for the extended version below.

## Extended Euclidean algorithm and modular inverse

The **extended** Euclidean algorithm doesn't just compute `gcd(a, b)` — it also finds integers
`x, y` such that `ax + by = gcd(a, b)` (Bézout's identity). That's the tool for computing a
**modular inverse**: the value `a⁻¹` such that `a · a⁻¹ ≡ 1 (mod m)`, which exists **if and only
if** `gcd(a, m) = 1`.

```python
def extended_gcd(a, b):
    if b == 0:
        return a, 1, 0
    g, x1, y1 = extended_gcd(b, a % b)
    return g, y1, x1 - (a // b) * y1

def mod_inverse(a, m):
    g, x, _ = extended_gcd(a, m)
    if g != 1:
        raise ValueError(f"no modular inverse: gcd({a}, {m}) = {g}, not 1")
    return x % m

mod_inverse(3, 11)   # 4, since 3*4 = 12 = 1 (mod 11)
```

Modular inverses are the mechanism behind computing an RSA private exponent from the public
exponent and totient, behind solving modular equations (`ax ≡ b (mod m)` becomes
`x ≡ a⁻¹·b (mod m)` once the inverse exists), and behind any "undo a multiplication in modular
space" problem, since division isn't directly defined mod `m`.

## Fast modular exponentiation

Computing `a^b mod m` for a large `b` (hundreds of digits, as in RSA) by first computing `a**b`
and then taking `% m` is catastrophic — `a**b` may be an astronomically large integer before the
modulus ever gets applied. Python's three-argument `pow` does square-and-multiply modular
exponentiation, keeping every intermediate value bounded by `m`:

```python
result = pow(a, b, m)          # fast, correct: O(log b) multiplications, each mod m
# result = (a ** b) % m        # correct but can be catastrophically slow/memory-heavy for large b
```

This single substitution — `pow(a, b, m)` instead of `(a ** b) % m` — is the most common
performance bug in hand-rolled crypto-adjacent code. Under the hood it's repeated squaring: to
compute `a^13`, note `13 = 1101₂` and combine `a^8 · a^4 · a^1`, each obtained by repeatedly
squaring and reducing mod `m`, giving `O(log b)` modular multiplications instead of `O(b)`.

## Primality testing: know the size before picking a method

```python
def is_prime_trial_division(n):
    if n < 2:
        return False
    for d in range(2, int(n**0.5) + 1):
        if n % d == 0:
            return False
    return True
```

Trial division up to `√n` is fine for numbers up to maybe a few million, but for crypto-sized
integers (hundreds of digits), `√n` is still astronomically large — trial division will never
finish. Practical code needs a probabilistic test:

```python
import random

def miller_rabin(n, k=20):
    if n < 2:
        return False
    for p in (2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31):
        if n % p == 0:
            return n == p
    d, r = n - 1, 0
    while d % 2 == 0:
        d //= 2
        r += 1
    for _ in range(k):
        a = random.randrange(2, n - 1)
        x = pow(a, d, n)
        if x in (1, n - 1):
            continue
        for _ in range(r - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False   # definitely composite
    return True             # probably prime (false-positive chance <= 4^-k)
```

Each round of Miller-Rabin either proves `n` composite for certain or leaves at most a 1-in-4
chance of a false "probably prime" — running `k=20` rounds drives that error probability down to
astronomically unlikely (`4⁻²⁰`), which is why every real crypto library uses this style of test
rather than a deterministic one for large numbers. In application code, just use
`sympy.isprime(n)` or a vetted library — hand-roll this only to understand what's happening
underneath.

**Naive Fermat primality testing is not safe as a substitute**: it checks `a^(n-1) ≡ 1 (mod n)`
for some base `a`, but **Carmichael numbers** (like 561) pass this test for *every* `a` coprime to
them despite being composite — a Fermat-only test can be fooled deterministically, which is
exactly why Miller-Rabin's extra structure (checking square roots of `1` along the way) is
necessary rather than a nice-to-have refinement.

## Linear Diophantine equations

`ax + by = c` has an integer solution **if and only if** `gcd(a, b)` divides `c` — and when it
does, the extended Euclidean algorithm hands you one directly (scale the Bézout coefficients).

```python
def solve_linear_diophantine(a, b, c):
    g, x0, y0 = extended_gcd(a, b)
    if c % g != 0:
        return None   # no integer solution exists
    scale = c // g
    x0, y0 = x0 * scale, y0 * scale
    # General solution family: x = x0 + (b/g)*t, y = y0 - (a/g)*t, for any integer t
    return x0, y0, b // g, a // g
```

This is the tool for scheduling/resource problems phrased as "find integers such that a linear
combination hits an exact target" (splitting a quantity into two coin/package denominations,
aligning two step sizes). **Nonlinear** Diophantine equations are a different, much harder world —
`a² + b² = c²` has infinite integer solutions (Pythagorean triples), while `aⁿ + bⁿ = cⁿ` has
*none* for integer `n > 2` (Fermat's Last Theorem, proven in 1994 after 350 years). Recognize the
difference: brute-forcing small cases of a nonlinear Diophantine equation to look for patterns is
reasonable exploratory code; trying to disprove a proven theorem by brute force is not a good use
of anyone's CPU time.

## Practical applications

**Check digits** (Luhn algorithm, used for credit card numbers): double every second digit from
the right, subtract 9 from any result over 9, sum everything, and the check digit is whatever
makes the total a multiple of 10 — pure modular arithmetic.

```python
def luhn_checksum(digits):
    total = 0
    for i, d in enumerate(reversed(digits)):
        if i % 2 == 1:
            d *= 2
            if d > 9:
                d -= 9
        total += d
    return total % 10

def is_valid_luhn(number_str):
    digits = [int(c) for c in number_str]
    return luhn_checksum(digits) == 0
```

ISBN-10 check digits work the same way with a different weighted-sum-mod-11 rule. Both exist to
catch the most common human transcription errors (single-digit typos, adjacent-digit transposition)
with a single extra digit — a direct, shippable use of modular arithmetic.

**Prime table sizes for hashing**: a hash table sized to a prime number spreads keys more evenly
when the hash function has arithmetic structure (e.g., linear combinations of key fields) — a
composite table size, especially a power of two, can interact badly with such a hash function and
concentrate collisions into a fraction of the slots. A power-of-two size (with bitmask indexing
instead of `%`) is fine, and faster, *when* the hash function itself is already high-quality and
well-mixed (e.g., a finalized 64-bit hash) — the prime-size advice matters most when the hash
function's own distribution is suspect.

**RSA in miniature** (for understanding, never for real security):

```python
p, q = 61, 53                      # toy primes -- real RSA uses 1000+ bit primes
n = p * q
phi = (p - 1) * (q - 1)
e = 17                              # public exponent, must be coprime with phi
d = mod_inverse(e, phi)             # private exponent

message = 42
ciphertext = pow(message, e, n)     # encrypt: modular exponentiation
plaintext = pow(ciphertext, d, n)   # decrypt: modular exponentiation with the inverse exponent
assert plaintext == message
```

This shows exactly where each primitive plugs in — totient for key generation, modular inverse to
derive the private key, fast modular exponentiation for both directions — but toy primes this
small are trivially factorable; never use hand-rolled RSA math for anything that needs to be
secure. Use a vetted library (and see `public-key-cryptography-and-tls-handshakes` for the
protocol layer).

## Common pitfalls

- **Computing `(a ** b) % m` instead of `pow(a, b, m)`** for large exponents — builds an
  astronomically large intermediate integer instead of staying bounded mod `m`.
- **Assuming a modular inverse always exists** — it exists only when `gcd(a, m) = 1`; calling
  `mod_inverse` without checking (or catching the failure) on a non-coprime pair is a silent
  correctness bug, not just a performance one.
- **Using trial division for primality testing on crypto-sized integers** — it will not finish in
  any practical time; use Miller-Rabin or a library's `isprime`.
- **Trusting a Fermat-only primality test** — Carmichael numbers pass it for every valid base
  while being composite; Miller-Rabin's extra square-root check is what closes that gap.
- **Assuming every Diophantine equation is solvable** — check `gcd(a,b) | c` for the linear case
  first; don't assume a nonlinear one has any solution at all (or that brute force will find one
  if none exists).
- **Hand-rolling RSA-style crypto for a real security boundary** — this skill's RSA example is for
  understanding the math, not a substitute for a vetted crypto library.

## Checklist

- [ ] Modular exponentiation uses `pow(a, b, m)`, never `(a ** b) % m`, for any exponent that
      isn't guaranteed small
- [ ] Modular inverse computation checks (or handles the failure of) `gcd(a, m) == 1` before
      assuming an inverse exists
- [ ] Primality testing method matches the number's size: trial division only for small numbers,
      Miller-Rabin (or `sympy.isprime`) for anything crypto-sized
- [ ] Any check-digit implementation (Luhn, ISBN, or similar) is tested against known valid and
      invalid numbers, not just derived from the formula and trusted
- [ ] Hash table sizing choice (prime vs power-of-two) is deliberate based on the hash function's
      own quality, not copied from habit
- [ ] Nothing security-sensitive uses hand-rolled RSA/crypto math from scratch — toy
      implementations are for understanding, production code uses a vetted library
