---
name: randomized-algorithm-design-patterns
description: Design, implement, or reason about algorithms that use randomness to get correctness or speed guarantees a deterministic version can't easily match — randomized quicksort/quickselect pivot selection, reservoir sampling from a stream of unknown length, Monte Carlo verification (e.g. Freivalds' algorithm for matrix multiplication), and probabilistic primality testing (Miller-Rabin). Use this when the user asks about Las Vegas vs. Monte Carlo algorithms, wants to avoid worst-case quicksort behavior on sorted/adversarial input, needs to sample k items fairly from a stream without knowing its length in advance, needs to verify a large computation's result cheaply and probabilistically, is implementing or debugging a primality test, or asks "how do randomized algorithms give guarantees." Also triggers on "bogosort," "one-sided error," "witness number," and "why does my quicksort degrade to O(n^2)."
---

# Randomized Algorithm Design Patterns

Randomized algorithms trade a small, quantifiable probability of a "bad" outcome for either a
much simpler algorithm or a much better expected running time than any known deterministic
approach. The value of this skill is knowing which of two families you're building, and how to
buy back as much confidence as you need by repeating the random choice — because "there's a
chance it's wrong" is only acceptable once you can say precisely how small that chance is, and
drive it arbitrarily lower.

## The two families: know which one you're building

- **Las Vegas**: always returns the *correct* result; the *running time* is what's random. If
  it terminates, trust the answer completely. Example: randomized quicksort — always sorts
  correctly, but a bad run of pivot choices can (rarely) take longer.
- **Monte Carlo**: has a *deterministic (or bounded) running time*; the *correctness* is what's
  probabilistic. Example: Miller-Rabin primality testing — always finishes quickly, but has a
  small, controllable chance of misclassifying a composite number as prime.

Any Las Vegas algorithm becomes a Monte Carlo algorithm the moment you cap its iteration count
and return a best-effort answer on timeout — this is a deliberate, common conversion, not a bug.
Decide up front which guarantee your caller actually needs (always-right-but-variable-time, or
bounded-time-with-quantifiable-error) and build to that contract explicitly — don't leave it
implicit in a `while True` loop that might never exit.

## Anti-pattern first: why bogosort teaches the right lesson

Permutation sort ("bogosort") repeatedly shuffles an array and checks whether it's sorted —
correct in the Las Vegas sense (it will eventually sort), catastrophic in expectation because
the search space is `n!` permutations:

```python
import numpy as np

def score(arr) -> int:
    """Count adjacent out-of-order pairs; 0 means sorted."""
    return sum(1 for i in range(len(arr) - 1) if arr[i] > arr[i + 1])

def bogosort(arr, rng, limit=None):
    """Las Vegas if limit is None (runs to completion); Monte Carlo (best-effort) otherwise."""
    arr = np.array(arr)
    best_score = score(arr)
    k = 0
    while best_score != 0 and (limit is None or k < limit):
        k += 1
        candidate = arr[np.argsort(rng.random(len(arr)))]
        s = score(candidate)
        if s < best_score:
            best_score, arr = s, candidate
    return arr, k
```

For `n=10`, expected iterations to find the one correct ordering out of `10! ≈ 3.6M` permutations
is measured in millions; for `n=20` it's `2.4×10^18` — no computer will finish in your lifetime.
**This is the load-bearing lesson**: randomization is a tool for buying a good expected case or
a probabilistic guarantee, not a substitute for algorithm design. If the "random guess and check"
version of a problem has a combinatorially exploding search space, randomizing it doesn't fix
that — you need a randomized algorithm that's structurally smarter (the ones below), not a random
search over the naive solution space.

## Randomized pivot selection: fixing quicksort's adversarial worst case

Standard quicksort's `O(n log n)` average case degrades to `O(n²)` — no better than bubble
sort — whenever the pivot choice (e.g., "always the first element") interacts badly with the
input's existing order, which happens on already-sorted or reverse-sorted input:

```python
import numpy as np

def quicksort_random(arr):
    """Randomized pivot: O(n log n) expected regardless of input order."""
    if len(arr) < 2:
        return arr
    pivot = arr[np.random.randint(0, len(arr))]
    low  = arr[arr < pivot]
    same = arr[arr == pivot]
    high = arr[arr > pivot]
    return np.hstack((quicksort_random(low), same, quicksort_random(high)))
```

Picking the pivot uniformly at random makes the *adversarial* case (a specific input crafted to
be slow) statistically indistinguishable from the *average* case — there's no longer a fixed
input that reliably triggers `O(n²)`, because the pivot sequence, not just the data, determines
the split. This is the textbook example of randomization buying a **worst-case-to-average-case
reduction**: same algorithm shape, provably better behavior under adversarial or already-ordered
input, at the cost of one random draw per partition.

**Randomized quickselect** (finding the k-th smallest element) uses the identical trick: pick a
random pivot, partition, then recurse into only the side containing the target rank — expected
`O(n)` instead of `O(n log n)` for a full sort, with the same worst-case-avoidance rationale.

## Reservoir sampling: fair sampling from a stream of unknown length

A common production need: pick `k` items uniformly at random from a stream (a log file, a
paginated API, a generator) **without knowing its length in advance** and without holding the
whole stream in memory. Reservoir sampling (Algorithm R) solves this in one pass:

```python
def reservoir_sample(stream, k: int, rng):
    """Return k items sampled uniformly at random from stream, in one pass, O(k) memory."""
    reservoir = []
    for i, item in enumerate(stream):
        if i < k:
            reservoir.append(item)
        else:
            j = rng.integers(0, i + 1)   # uniform over [0, i]
            if j < k:
                reservoir[j] = item
    return reservoir
```

The `i`-th item (0-indexed) replaces a random slot with probability `k/(i+1)`, which works out
to exactly `k/n` for every item once the stream of length `n` is exhausted — provably uniform,
not approximately. Reach for this whenever "sample k rows from a table/log/stream" meets
"I don't want to load it all into memory to call `random.sample`."

## Monte Carlo verification: Freivalds' algorithm

Checking whether `AB = C` for `n×n` matrices by actually computing `AB` costs `O(n³)`. Freivalds'
algorithm checks it in `O(n²)` by testing a random linear projection instead of the full product:

```python
import numpy as np

def freivalds_check(A, B, C, rng, rounds=20) -> bool:
    """Probabilistically verify A @ B == C in O(rounds * n^2) instead of O(n^3)."""
    n = A.shape[0]
    for _ in range(rounds):
        r = rng.integers(0, 2, size=n)          # random 0/1 vector
        if not np.array_equal(A @ (B @ r), C @ r):
            return False                         # definitely AB != C
    return True                                   # AB == C with high probability
```

The algorithm is **one-sided**: if `AB ≠ C`, each round catches it with probability ≥ 1/2, so it
never wrongly says "not equal" when they are equal, but it can (rarely) say "equal" when they
aren't. Because rounds are independent, the false-positive probability shrinks geometrically:
`(1/2)^rounds`. This is the generalizable pattern for Monte Carlo verification — **when checking
a result exactly is expensive, check a cheap randomized property of it many times and let the
failure probability compound down to whatever you need** (20 rounds → under one-in-a-million).

## Monte Carlo verification: Miller-Rabin primality

Same shape as Freivalds', applied to primality: a Fermat-like modular check that's cheap to run
and, when repeated with independently random witnesses, drives the false-positive rate down
geometrically.

```python
def is_probable_prime(n: int, rounds: int, rng) -> bool:
    """Miller-Rabin: one-sided error. False => definitely composite.
    True  => prime with probability >= 1 - 4**-rounds (usually far better)."""
    if n == 2:
        return True
    if n < 2 or n % 2 == 0:
        return False
    # write n - 1 = 2^s * d with d odd
    s, d = 0, n - 1
    while d % 2 == 0:
        s, d = s + 1, d // 2
    for _ in range(rounds):
        a = rng.integers(2, n - 1)
        x = pow(a, d, n)
        if x in (1, n - 1):
            continue
        for _ in range(s - 1):
            x = pow(x, 2, n)
            if x == n - 1:
                break
        else:
            return False   # a is a witness: n is definitely composite
    return True
```

Key properties worth internalizing because they generalize to every Monte Carlo algorithm you'll
design:

- **One-sided error again.** A "composite" verdict is always correct (a *witness* number was
  found). A "probably prime" verdict has a bounded chance of being wrong (an unlucky *non-witness*
  was drawn). Never symmetric — know which direction your algorithm's error can go.
- **Worst-case false-positive rate is `4^-rounds`**, but for most `n` the real rate is far lower.
  Pick `rounds` based on the worst case you're willing to defend, not the typical case.
- **It's asymptotically far cheaper than brute force**: Miller-Rabin is `O(k log³ n)` for `k`
  rounds vs. brute-force trial division's `O(√n)` — brute force becomes unusable well before
  15-digit numbers; Miller-Rabin stays fast into the hundreds of digits, which is why it (or a
  close relative) underlies real-world RSA key generation.

## Common pitfalls

- **Randomizing the wrong layer.** Bogosort-style "randomly guess a full solution and check it"
  almost never helps if the solution space is combinatorially large — randomize a *choice inside*
  a structurally sound algorithm (a pivot, a witness, a projection vector), not the entire
  output.
- **Not stating which error direction is possible.** If you can't say "this only ever fails by
  saying X when the truth is Y, never the reverse," you don't yet understand your algorithm's
  error model well enough to ship it.
- **Picking a round/rejection count without connecting it to an actual probability bound.**
  "Run it a few times" is not a specification; "20 rounds bounds false positives below 1e-6" is.
- **Forgetting Las Vegas algorithms need a timeout/cap in production.** An unbounded `while True`
  waiting for a correct-but-random-time result is a latent hang; decide the fallback behavior
  (retry, deterministic algorithm, error) before it ships.
- **Using `random`/an LCG instead of a well-distributed source for the random choices.** A
  pivot, witness, or projection vector drawn from a poor-quality generator can reintroduce
  exactly the correlations the randomization was meant to break — see the
  `prng-selection-and-testing` skill for source selection.

## Checklist

- [ ] Identified whether the algorithm is Las Vegas (variable time, always correct) or Monte
      Carlo (bounded time, probabilistically correct) — and documented that explicitly
- [ ] For Monte Carlo algorithms, stated which direction the error can go (one-sided vs.
      two-sided) and the false-positive/false-negative bound as a function of repetitions
- [ ] Randomization is applied to a structural choice inside a sound algorithm (pivot, witness,
      projection), not as a brute-force guess-and-check over the whole output space
- [ ] Las Vegas algorithms have an explicit cap/timeout and defined fallback behavior in
      production code
- [ ] Reservoir sampling (or an equivalent one-pass technique) is used instead of
      "load everything then `random.sample`" when the stream size is unknown or large
- [ ] The randomness source feeding these algorithms has been chosen deliberately, not left as
      whatever was already imported
