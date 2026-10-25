---
name: shors-algorithm-and-post-quantum-cryptography
description: Reason accurately about Shor's algorithm — why it factors integers exponentially faster than any known classical algorithm, how it reduces factoring to period-finding solved via the quantum Fourier transform, why it threatens RSA specifically, and why it is not a near-term practical threat. Use this whenever the user asks how Shor's algorithm works, whether quantum computers can "break encryption," what post-quantum cryptography is and why it's needed now, is evaluating a vendor/media claim about quantum computers cracking RSA or Bitcoin, or wants the classical-vs-quantum split of the algorithm (order-finding, modular exponentiation, continued fractions vs. the quantum Fourier transform). Also triggers on "period finding," "quantum Fourier transform," "RSA quantum vulnerability," and "NIST post-quantum standards."
---

# Shor's Algorithm and Post-Quantum Cryptography

Shor's algorithm factors large composite integers in polynomial time on a quantum computer,
versus the best known classical algorithms, which take sub-exponential (but still infeasible for
large enough keys) time. This is the algorithm that makes quantum computing a headline
cryptography story — RSA's security rests entirely on factoring being classically hard — but the
gap between "this algorithm exists and is proven correct" and "a quantum computer can break your
TLS certificate today" is enormous and worth being precise about, in both directions.

## What actually gets broken, and why

RSA's security assumption is: given `N = p·q` for two large primes `p` and `q`, no known
classical algorithm can recover `p` and `q` from `N` alone in a practical amount of time. Shor's
algorithm doesn't attack RSA's math directly — it solves the **integer factorization problem** in
general, and RSA just happens to be built on the hardness of that specific problem. Any
cryptosystem whose security reduces to factoring (or to the closely related discrete-logarithm
problem, which a variant of Shor's algorithm also solves — this is what breaks Diffie-Hellman and
elliptic-curve cryptography too) is threatened the same way. Symmetric cryptography (AES) is
**not** broken by Shor's algorithm — its threat there is a different, much smaller quadratic
speedup from Grover's algorithm (see `grovers-algorithm-and-quantum-search`), addressed by simply
doubling key length, not by replacing the scheme.

## The classical-to-quantum reduction: factoring → order-finding

Shor's algorithm is a **hybrid** algorithm — classical computation on both ends, with a quantum
subroutine doing the one piece classical computers can't do efficiently.

**Classical part 1 (setup):** to factor `N`, pick a random integer `a` coprime to `N` (verified
via the classical, efficient Euclidean algorithm — if `gcd(a, N) ≠ 1`, you've accidentally already
found a factor and you're done). The problem now reduces to finding the **order** `r` of `a`
modulo `N` — the smallest positive integer such that:

```
a^r ≡ 1 (mod N)
```

Finding `r` classically requires checking modular exponentials in the worst case up to
exponentially many candidate periods — this is the genuinely hard step, and it's a specific
instance of the same "find the period of a function" problem Simon's algorithm demonstrated a
quantum advantage for (see `oracle-based-quantum-algorithms`).

**Classical part 2 (finish):** given `r` (if it's even — if not, pick a new `a` and retry), compute:

```
gcd(a^(r/2) − 1, N)   and   gcd(a^(r/2) + 1, N)
```

With high probability, at least one of these gcds is a nontrivial factor of `N`. If not (there are
a couple of known failure conditions), retry with a new random `a`. In practice, this succeeds
often enough that fewer than 10 attempts are needed on average to fully factor `N`.

## The quantum part: period-finding via the quantum Fourier transform

The quantum subroutine finds `r` (or, more precisely, solves the closely related periodic
structure of `f(x) = aˣ mod N`) using two registers and the **quantum Fourier transform (QFT)**:

```
1. Prepare register v (n qubits) in |0>, register w (n qubits) in |0>
2. Apply H^n to v -> uniform superposition over all N = 2^n values
3. Apply a reversible circuit computing f(x) = a^x mod N, writing the result into w:
     sum_x |x>|0>  ->  sum_x |x>|a^x mod N>
4. Measure register w, collapsing register v to a superposition of only those
   x consistent with the measured f(x) value — i.e. a superposition of |b>, |b+r>,
   |b+2r>, ... spaced exactly r apart
5. Apply the QFT to register v
6. Measure register v -> yields (with useful probability) an integer close to a
   multiple of N/r
7. Classically, continued-fraction expansion recovers r from that measurement
```

The QFT is the piece with no efficient classical analog for this purpose: it converts the
period-`r` spacing hidden in register `v`'s superposition into a directly measurable numeric
signal (a value clustering near multiples of `N/r`), the same way a classical Fourier transform
turns a periodic time-domain signal into sharp spikes in the frequency domain. This is the exact
same period-extraction shape Simon's algorithm uses, generalized from bitwise-XOR periodicity to
full modular-arithmetic periodicity — which is why Simon's algorithm is usually taught first, as
conceptual preparation for this step.

## Why this isn't a near-term practical threat

As of the most recent public estimates, breaking real-world RSA key sizes (2048-bit or larger)
with Shor's algorithm requires a quantum computer with many thousands of high-quality *logical*
qubits, sustained through the full period-finding circuit — which, accounting for the physical
qubits needed per logical qubit under current error-correction overhead (see
`quantum-error-correction-and-nisq-limitations`), is far beyond any publicly known hardware today.
"Shor's algorithm exists and is correct" is not the same claim as "your encrypted traffic is at
risk right now" — treat vendor or media claims of "quantum computers breaking encryption" with
that distinction firmly in mind; check specifically what qubit count, logical-qubit fidelity, and
circuit depth the claim assumes before taking it at face value.

## Why this motivates action anyway: post-quantum cryptography

The credible risk isn't "today," it's **"harvest now, decrypt later"**: an adversary can record
encrypted traffic today and decrypt it once a sufficiently capable quantum computer exists years
from now — a real concern for any data that needs confidentiality on a multi-year or multi-decade
horizon. This is the actual motivation behind **post-quantum cryptography (PQC)**: cryptographic
schemes believed to be hard for *both* classical and quantum computers (typically based on
lattice problems, hash-based signatures, or other structures with no known Shor-style reduction).
NIST finalized its first PQC standards in 2024, and migrating long-lived, sensitive systems to
PQC-ready algorithms (or hybrid classical+PQC schemes) is a live, present-day engineering
recommendation — independent of exactly when a cryptographically relevant quantum computer
arrives.

## Common pitfalls

- **Saying "quantum computers can break encryption" without qualification.** Symmetric
  cryptography (AES) is only quadratically weakened (Grover), not broken; only
  factoring/discrete-log-based public-key schemes (RSA, classic Diffie-Hellman, ECC) are
  threatened by Shor's algorithm specifically.
- **Treating Shor's algorithm as purely quantum.** It's a hybrid: classical setup (pick `a`, check
  gcd), a quantum period-finding subroutine (QFT), and classical post-processing (continued
  fractions, final gcd computation) — most of the algorithm's total logic is classical.
- **Confusing "Shor's algorithm is proven to work" with "current hardware can run it at
  cryptographically relevant scale."** The algorithm's correctness and the hardware's current
  capability are two entirely separate claims — the second is nowhere close to true yet.
  ("Quantum computers already factored a huge RSA number" headlines are typically about
  small-integer demonstrations with a pre-chosen easy factorization, not general-purpose attacks
  on real key sizes.)
- **Assuming migrating to post-quantum cryptography can wait until quantum computers "arrive."**
  The harvest-now-decrypt-later threat model means data needing long-term confidentiality is
  already at risk from future decryption of traffic captured today.
- **Missing that the order `r` must be even (and that `a^(r/2) ≢ −1 mod N`) for the final gcd step
  to work.** If not, the algorithm doesn't fail outright — it just requires retrying with a new
  random `a`, which is why the algorithm is described as succeeding "with high probability," not
  deterministically on the first attempt.

## Checklist

- [ ] Claims of "Shor's algorithm breaks X" are scoped correctly: factoring- or discrete-log-based
      public-key schemes, not symmetric ciphers or hash functions
- [ ] The algorithm is described as hybrid — classical setup and post-processing surrounding one
      quantum period-finding subroutine — not as an end-to-end quantum computation
- [ ] Any practicality claim states the qubit count / error-correction assumptions it depends on,
      rather than treating algorithmic existence as equivalent to present-day feasibility
- [ ] Long-term-confidentiality data is flagged for post-quantum migration consideration
      regardless of current quantum hardware timelines (harvest-now-decrypt-later)
- [ ] The role of the QFT is described as converting hidden periodicity into a measurable
      numeric signal, with the final factor extraction (continued fractions, gcd) done classically
