---
name: prng-selection-and-testing
description: Choose the right source of randomness for a task (fast simulation vs. reproducible tests vs. cryptography vs. space-filling search initialization) and verify it's actually behaving randomly enough. Use this whenever the user asks "which random number generator should I use," is choosing between numpy's default_rng, random, secrets, os.urandom, or a hardware RNG, needs a reproducible seeded RNG for tests or experiments, is building an abstraction that swaps randomness sources behind one interface, is initializing a search/optimization/sampling routine and wonders whether to use quasirandom (Sobol/Halton) sequences instead of pseudorandom ones, or is asking how to check whether a generator's output is "random enough" (entropy, chi-square, correlation tests). Also triggers on "is this PRNG good enough," "random seed reproducibility," "quasirandom vs pseudorandom," and "don't use random.random() for security."
---

# PRNG Selection and Testing

There is no single "best" randomness source — there's only the right source for what you're
about to do with it. Treating "randomness" as one interchangeable thing is how insecure
tokens get generated with `random.random()` and how flaky tests get "reproducible" bugs that
vanish when someone reruns them. This skill covers picking the right source, building one
abstraction that lets you swap sources without touching call sites, and gaining calibrated
confidence that a source's output is actually random enough for the job.

## Decision: which source for which job

| Use case | Use | Never use |
|---|---|---|
| General simulation, ML init, games, art | `numpy.random.Generator(PCG64())` (NumPy's default `default_rng()`) or `MT19937` | — |
| Cryptography, tokens, session IDs, password reset links | `secrets` module, `os.urandom`, or a CSPRNG seeded from OS entropy | `random`, any LCG, MT19937 (its state is invertible from 624 outputs) |
| Reproducible tests / experiments that must replay exactly | Any PRNG **with an explicit fixed seed**, stored/logged alongside the result | An unseeded generator, or "whatever the OS gives us" |
| Space-filling initialization (swarm search start points, low-discrepancy sampling, numerical integration) | Quasirandom sequences (Halton, Sobol via `scipy.stats.qmc`) | A single pseudorandom stream reused across dimensions |
| Hedging against a compromised or biased software PRNG | Hybrid: reseed a CSPRNG from a hardware source (`/dev/urandom`, CPU `RDRAND`) periodically | Trusting a hardware instruction alone without a fallback (not all CPUs support it) |
| Teaching/demoing what a PRNG *is* | A hand-rolled LCG (e.g., MINSTD) — small, inspectable, deliberately weak | Ever, in anything that ships |

The recurring mistake is reaching for whatever's convenient (`random.random()`) regardless of
which column the task is actually in. `random` and a plain LCG are **statistically weak and
have small/invertible internal state** — fine for a game or a book's toy examples, unacceptable
the moment output influences security or money.

## Why MT19937 and LCGs are not cryptographic

A linear congruential generator is a single equation:

```
x[i+1] = (a * x[i] + c) mod m
```

Given consecutive outputs, `a`, `c`, and `m` are usually easy to recover or guess, at which point
every future output is predictable. MT19937 (NumPy's old default, Python's `random`) is
statistically excellent for simulation — but its entire internal state can be reconstructed
from 624 consecutive 32-bit outputs, after which every future value is known. Neither fact
matters for a Monte Carlo simulation. Both are disqualifying for anything an adversary can
observe and would benefit from predicting.

PCG64 (NumPy's current default) has a much larger period (2^128) and better statistical
properties than MT19937, but it is *still not a CSPRNG* — use `secrets`/`os.urandom` for
anything security-sensitive.

## Quasirandom sequences: when "less random" is better

A quasirandom (low-discrepancy) sequence, like a Halton sequence, is fully deterministic but
fills space more evenly than a pseudorandom sequence of the same length — no clumps, no gaps.
Reach for one whenever *even coverage* matters more than *unpredictability*: initializing
particle positions for a swarm optimizer, seeding a numerical integration grid, or picking
initial hyperparameter trial points before a smarter search takes over.

A Halton sequence for prime base `b` is built by reversing the base-`b` digits of the index:

```python
def halton(index: int, base: int) -> float:
    """Return the index-th element of the Halton sequence for the given prime base."""
    result, f = 0.0, 1.0 / base
    i = index
    while i > 0:
        result += f * (i % base)
        i //= base
        f /= base
    return result
```

**The trap**: sampling `(halton(i, 2), halton(i+1, 2))` as an (x, y) pair does *not* fill 2D
space — consecutive outputs of the same base are trivially related to each other, so the pairs
fall on a predictable curve. Use a **different prime base per dimension**:

```python
xs = [halton(i, 2) for i in range(n)]   # x-coordinates: base 2
ys = [halton(i, 3) for i in range(n)]   # y-coordinates: base 3 (must differ from x's base)
```

For anything beyond 2–3 dimensions, use `scipy.stats.qmc.Sobol` or `Halton` instead of
hand-rolling — Sobol sequences scale to higher dimensions with better uniformity guarantees.

## Build one randomness-source abstraction, not scattered `np.random` calls

The single highest-leverage pattern here: wrap every randomness source behind one class with a
uniform call signature, so swapping "which RNG" for a test, a benchmark, or a security review is
a one-line change instead of a grep-and-replace across the codebase.

```python
import numpy as np

class RandomSource:
    """Uniform interface over multiple randomness sources.

    mode: 'float' -> [low, high), 'int' -> [low, high), 'byte' -> uint8, 'bit' -> 0/1
    kind: 'pcg64' | 'mt19937' | 'urandom' | a path to a file of pre-recorded bytes (for replay)
    """
    def __init__(self, mode="float", kind="pcg64", seed=None, low=0.0, high=1.0):
        self.mode, self.low, self.high, self.kind = mode, low, high, kind
        self._file = None
        if kind == "pcg64":
            self._gen = np.random.Generator(np.random.PCG64(seed))
        elif kind == "mt19937":
            self._gen = np.random.Generator(np.random.MT19937(seed))
        elif kind == "urandom":
            self._gen = None
        elif Path(kind).exists():          # replay a fixed byte stream: perfectly reproducible
            self._file = open(kind, "rb")
        else:
            raise ValueError(f"unknown randomness source: {kind}")

    def _uniform(self, n: int) -> np.ndarray:
        if self._file is not None:
            b = self._file.read(4 * n) or (self._file.seek(0) or self._file.read(4 * n))
            return np.frombuffer(bytearray(b), dtype="uint32") / (1 << 32)
        if self.kind == "urandom":
            with open("/dev/urandom", "rb") as f:
                b = f.read(4 * n)
            return np.frombuffer(bytearray(b), dtype="uint32") / (1 << 32)
        return self._gen.random(n)

    def random(self, n: int = 1):
        v = self._uniform(n)
        if self.mode == "float":
            v = (self.high - self.low) * v + self.low
        elif self.mode == "int":
            v = ((self.high - self.low) * v).astype("int64") + self.low
        elif self.mode == "byte":
            v = np.floor(256 * v).astype("uint8")
        elif self.mode == "bit":
            v = np.floor(v + 0.5).astype("uint8")
        return v[0] if n == 1 else v
```

Why this earns its keep beyond DRY:

- **Reproducible tests for free.** Point `kind` at a small pre-recorded byte file during CI and
  every test gets the exact same "random" sequence forever, independent of library version
  upgrades changing a PRNG's internal algorithm.
- **One place to swap for a security review.** If someone later needs a cryptographically
  secure path, there's one method (`_uniform`) to change, not every call site.
- **One place to log the seed.** Reproducing a bug from a random test failure means logging
  `kind` + `seed` once, in the constructor, not hunting through the code for every `np.random`
  call that might have contributed.

## Testing whether a source is "random enough"

You cannot *prove* a sequence is random — you can only fail to find evidence that it isn't.
Calibrate the bar to the stakes:

- **Quick sanity check (simulation/game stakes)**: the Linux `ent` tool (`apt-get install ent`,
  `brew install ent` on macOS) run against a file of raw bytes from the source reports entropy
  (bits/byte, want ≈8.0), a chi-square percentage (5–95% is fine; outside that range is
  suspicious), the arithmetic mean (want ≈127.5 for bytes), a Monte Carlo π estimate (should be
  close to 3.14159), and serial correlation (want ≈0.0). This is a smoke test, not a proof.
- **Statistical rigor (research, publishing results)**: run a real test suite — TestU01,
  PractRand, or Dieharder — which applies dozens of tests far beyond frequency counts,
  including short- and long-range correlation structure.
- **Security stakes**: don't test statistical randomness at all as your bar — use a source that
  is *designed and reviewed* as a CSPRNG (`secrets`, `os.urandom`, a vetted hardware RNG path).
  Passing `ent` says nothing about resistance to an adversary who's trying to predict your
  output; a CSPRNG's design and review history is the actual guarantee.

A chi-square/entropy pass is evidence toward "good enough for simulation," never evidence of
cryptographic strength — the two questions are unrelated.

## Common pitfalls

- **Using `random.random()` or a hand-rolled LCG for tokens, keys, or anything with a security
  boundary.** Small/invertible state means an observer who sees enough output can predict the
  rest.
- **Forgetting to fix (and log) the seed for anything that must be reproducible** — a flaky test
  that "randomly" fails once a month is nearly always an unseeded RNG plus insufficient logging
  of the seed that produced the failure.
- **Reusing a fixed seed in production** where unpredictability is actually wanted — the
  double-edged sword of seeding: great for tests, a bug in a service that should never repeat
  the same "random" sequence for every user/session.
- **Plotting consecutive quasirandom outputs as coordinates in >1 dimension** without a distinct
  prime base per dimension — the classic "why do all my points fall on a curve" bug.
- **Confusing a statistical pass (`ent`, chi-square) with cryptographic security.** They test
  different, unrelated properties.
- **Ignoring precision loss** — a 32-bit generator (LCG/MINSTD) dividing into `[0,1)` only ever
  produces ~2^31 distinct floats, far fewer than a 64-bit float can represent; fine for most
  simulations, a real limitation if you need fine-grained continuous coverage.

## Checklist

- [ ] The chosen source matches the use-case row above (simulation vs. crypto vs. reproducible
      tests vs. space-filling init) — not just "whatever was already imported"
- [ ] Anything security-sensitive uses `secrets`/`os.urandom`/a vetted CSPRNG, never `random` or
      a custom LCG/PCG/MT
- [ ] Reproducible runs (tests, published experiments) fix and log the seed alongside the result
- [ ] Quasirandom sequences use a distinct prime base per dimension, or `scipy.stats.qmc` for
      >2 dimensions
- [ ] If randomness quality was validated, note *how* (a quick `ent` smoke test vs. a full
      TestU01/PractRand run) so future readers know how much confidence that check actually buys
- [ ] A single abstraction (class/module) owns "get me N random values," so switching sources
      later doesn't require a codebase-wide find-and-replace
