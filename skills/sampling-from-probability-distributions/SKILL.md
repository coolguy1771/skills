---
name: sampling-from-probability-distributions
description: Draw samples from an arbitrary discrete or continuous probability distribution in code — a custom PMF, a PDF with no built-in sampler, or a posterior you can only evaluate up to a normalizing constant. Use this when the user needs to sample from a weighted/categorical distribution efficiently, wants inverse transform sampling for a distribution with a known CDF, needs rejection sampling for a PDF with no closed-form inverse, is implementing or debugging Markov Chain Monte Carlo / Metropolis-Hastings, asks "how do I sample from this distribution in Python," or is choosing between sampling algorithms for performance (many repeated draws) vs. simplicity. Also triggers on "weighted random choice," "importance sampling," "MCMC," "Metropolis-Hastings," and "sample from a PDF I can't invert."
---

# Sampling from Probability Distributions

Every distribution-sampling problem reduces to one question: how expensive is it to turn a
uniform `[0,1)` random number into a sample from the distribution you actually want? The answer
determines which of four techniques to reach for — and picking the wrong one either produces
wrong output (silently) or burns orders of magnitude more compute than necessary.

## Discrete distributions: three algorithms, one correctness bar

Given a PMF (a set of weights over a finite set of outcomes), all three algorithms below produce
*statistically identical* output — the only difference is speed, which matters once you're
drawing more than a handful of samples.

**Sequential search (inversion by subtraction)** — simplest to write, slowest to run:

```python
def sample_sequential(probs, rng) -> int:
    """probs: normalized PMF (sums to 1). O(k) per sample, k = number of outcomes."""
    u = rng.random()
    k = 0
    while u > 0:
        u -= probs[k]
        k += 1
    return k - 1
```

**Reorder for a quick win**: sort `probs` descending once, sample with the same loop, then map
the returned index back through the sort permutation. The most likely outcome now costs one
pass instead of potentially scanning the whole distribution:

```python
import numpy as np

idx = np.argsort(probs)[::-1]      # indices sorted by decreasing probability
sorted_probs = probs[idx]
raw = sample_sequential(sorted_probs, rng)
outcome = idx[raw]                  # map back to the true index
```

**Fast Loaded Dice Roller (FLDR)** — near-optimal, `O(1)` amortized per sample after an `O(k)`
one-time preprocessing step. Use the reference implementation (`pip3 install fldr`, or the
`fldr.py` file from the original paper's repo) rather than reimplementing it — the construction
is intricate and the whole point is that a careful implementation beats naive approaches by
close to an order of magnitude:

```python
from fldr import fldr_preprocess_int, fldr_sample

weights = [1, 1, 3, 4, 5, 1, 7, 4, 3]     # integer weights, need not be normalized
sampler = fldr_preprocess_int(weights)     # pay this cost once
samples = [fldr_sample(sampler) for _ in range(100_000)]   # then O(1) each
```

**Decision rule**: reach for sequential/reorder for a one-off or tiny distribution where clarity
beats speed; reach for FLDR the moment you're sampling the same PMF many times (Monte Carlo
inner loops, simulations, game logic) — the preprocessing cost amortizes almost immediately and
the runtime gap widens with sample count.

For a 2D discrete distribution (e.g., sampling pixel positions weighted by image intensity),
flatten to 1D, sample an index with any of the above, then unravel back to (row, col) — don't
build a 2D sampler from scratch.

## Continuous distributions: inverse transform

If you know the CDF `F(x)` **and can invert it in closed form**, inverse transform sampling is
the cheapest possible technique — one uniform draw, one function evaluation, done:

```python
import numpy as np

def sample_exponential(rate: float, n: int, rng) -> np.ndarray:
    """F(x) = 1 - e^(-rate*x)  =>  F^-1(u) = -ln(1-u)/rate, or equivalently -ln(u)/rate
    since u and 1-u are both uniform on [0,1)."""
    u = rng.random(n)
    return -np.log(u) / rate
```

The general recipe: derive `F(x) = ∫f(t)dt`, solve `u = F(x)` for `x` to get `F⁻¹(u)`, then draw
`u ~ Uniform[0,1)` and return `F⁻¹(u)`. This only works when both steps are tractable in closed
form — most arbitrary PDFs fail one or the other, which is exactly when to reach for rejection
sampling instead.

## Continuous distributions: rejection sampling

Works for **any** PDF `q(x)` you can evaluate pointwise, even with no closed-form CDF, as long as
you can find a proposal `p(x)` you know how to sample from and a constant `c` such that
`q(x) ≤ c·p(x)` for all `x` (i.e., the scaled proposal envelopes the target everywhere):

```python
def sample_rejection(q, propose, c: float, n: int, rng) -> list[float]:
    """q: target PDF (callable). propose: () -> x, a sample from p(x).
    c: constant s.t. q(x) <= c * p(x) for all x in the support."""
    samples = []
    trials = 0
    while len(samples) < n:
        trials += 1
        x = propose()
        u = rng.random()
        if u * c * proposal_density(x) <= q(x):     # accept iff the "dart" lands under q(x)
            samples.append(x)
    return samples
```

Think of it as throwing darts uniformly under the envelope `c·p(x)` and keeping only the ones
that also land under `q(x)` — the accepted x-coordinates are distributed exactly as `q(x)`.

**The only design decision that matters is how tight the envelope is.** Acceptance rate equals
roughly the ratio of the area under `q(x)` to the area under `c·p(x)`; a loose envelope (large
`c`, or a proposal shape that doesn't resemble the target) can mean rejecting 80–90%+ of
candidates, burning enormous amounts of compute for the accepted fraction. Before committing to
a proposal, sanity-check the acceptance rate on a small run — if it's in the single digits,
find a tighter-fitting proposal (a normal instead of a uniform envelope is a common fix when the
target has a peaked, bell-like shape) rather than just running more trials.

Rejection sampling extends to multiple dimensions, but the acceptance rate degrades quickly as
dimensionality grows (the curse of dimensionality shrinks the fraction of the bounding volume
that's actually under the target density) — past a handful of dimensions, MCMC below is usually
the more practical choice.

## When neither works: Markov Chain Monte Carlo (Metropolis-Hastings)

Use MCMC when you can only evaluate the target distribution **up to a normalizing constant**
(the overwhelmingly common case for Bayesian posteriors: `posterior ∝ likelihood × prior`, and
computing the true normalizing constant requires an intractable integral). Metropolis-Hastings
sidesteps ever needing that constant because it only ever looks at *ratios* of the target
density:

```python
def metropolis_hastings(log_target, x0: float, sigma: float, n: int, rng) -> list[float]:
    """log_target(x): log of the (possibly unnormalized) target density pi(x).
    Random-walk proposal x' ~ N(x, sigma). Returns n samples from pi(x) once the
    chain has mixed (discard an initial burn-in from the returned list)."""
    x = x0
    samples = []
    for _ in range(n):
        x_proposed = rng.normal(x, sigma)
        log_ratio = log_target(x_proposed) - log_target(x)   # normalizing constants cancel
        if np.log(rng.random()) < min(0.0, log_ratio):
            x = x_proposed
        samples.append(x)
    return samples
```

Work in **log-space** for the acceptance ratio (`log_target` rather than `target`, comparing
`log(u) < log_ratio` instead of `u < ratio`) — target densities in real problems routinely
underflow to 0 in linear space long before they're actually negligible, and log-space avoids
that silently-wrong-answer failure mode entirely.

Practical knobs and their failure signatures:

- **Proposal width (`sigma`) too small** → the chain barely moves, acceptance rate near 100%,
  but it takes forever to explore the full distribution (high autocorrelation between samples).
- **Proposal width too large** → almost every proposal is rejected, the chain gets stuck at the
  same value for long runs.
- **A healthy random-walk Metropolis chain** typically targets roughly 20–50% acceptance —
  tune `sigma` to land in that range before trusting the output.
- **Burn-in**: discard samples from the beginning of the chain before it has reached the
  stationary distribution (inspect a trace plot of sample value vs. iteration; burn-in is over
  once it looks like it's wandering around a stable range rather than trending from the start
  value).
- **Convergence is a Markov chain property, not a guarantee** — the chain provably reaches the
  target distribution *in the limit*; in practice, always run multiple chains from different
  starting points and check they agree before trusting the result on anything that matters.

## Choosing among the four techniques

| You have | Use |
|---|---|
| A discrete PMF, sampled many times | FLDR (or reorder-then-sequential for one-offs) |
| A PDF with a closed-form, invertible CDF | Inverse transform |
| A PDF you can evaluate but can't invert its CDF, low dimensions | Rejection sampling |
| A target known only up to a normalizing constant, or high-dimensional | MCMC (Metropolis-Hastings, or a library like PyMC/Stan for anything production-grade) |

## Common pitfalls

- **Reinventing FLDR-quality performance by hand.** A naive sequential search is fine for a
  handful of draws; at scale it's needlessly `O(k)` per sample when `O(1)` amortized is available.
- **Rejection sampling with a proposal that doesn't resemble the target's shape.** A uniform
  envelope over a sharply peaked target can mean rejecting the overwhelming majority of
  candidates — measure the acceptance rate, don't assume it's fine.
- **Computing the MH acceptance ratio in linear space** instead of log-space, causing silent
  underflow-to-zero on realistic likelihoods.
- **Trusting a single MCMC chain.** Run several from different starting points; disagreement
  between them is the clearest signal that the chain hasn't converged or `sigma` needs tuning.
- **Not discarding burn-in**, which biases every downstream summary statistic toward the
  arbitrary starting value.
- **Using inverse transform on a CDF that isn't actually invertible in closed form** and
  papering over it with a numerical root-find per sample — correct, but often orders of
  magnitude slower than switching to rejection sampling or MCMC outright.

## Checklist

- [ ] Matched the technique to what's known about the distribution (closed-form invertible CDF →
      inverse transform; evaluable PDF, low-dim → rejection; unnormalized/high-dim → MCMC;
      discrete PMF sampled repeatedly → FLDR)
- [ ] For rejection sampling, measured the acceptance rate and tightened the proposal if it's
      in the single digits
- [ ] For MCMC, worked in log-space for the acceptance ratio
- [ ] For MCMC, checked acceptance rate lands roughly in 20–50% and tuned proposal width if not
- [ ] For MCMC, discarded burn-in and cross-checked multiple chains from different starting
      points before trusting the result
- [ ] For a discrete distribution sampled many times, used FLDR (or an equivalent `O(1)`
      amortized structure) instead of a fresh `O(k)` scan per sample
