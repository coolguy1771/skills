---
name: swarm-and-evolutionary-optimization
description: Optimize a black-box or non-differentiable objective function using population-based randomized search — particle swarm optimization (PSO), differential evolution (DE), genetic algorithms (GA), Jaya, or simple random optimization (RO) — instead of gradient-based methods. Use this when the user needs to fit parameters to data without a usable gradient, wants to optimize a function with no clean derivative (discontinuous, noisy, combinatorial, or code-shaped like genetic programming), asks "which swarm algorithm should I use" or "PSO vs. genetic algorithm vs. differential evolution," is designing an objective/fitness function and bounds for a metaheuristic search, or needs to evolve a program/expression from data (genetic programming, symbolic regression). Also triggers on "particle swarm," "differential evolution," "objective function," "premature convergence," and "swarm intelligence."
---

# Swarm and Evolutionary Optimization

Every algorithm in this family shares one shape: scatter candidate solutions (particles/
individuals) through the search space, score each with an objective function, decide where they
go next, and repeat until a stopping condition. **What differs between algorithms is entirely the
"decide where they go next" rule** — and different rules trade off convergence speed, tuning
burden, and susceptibility to getting stuck. Reach for this family whenever gradient descent
isn't available or doesn't apply: no usable derivative, a discrete/combinatorial search space, a
noisy or discontinuous objective, or the "parameters" being searched are actually a program's
structure (genetic programming).

## When to reach for this instead of gradient-based optimization

Use a swarm/evolutionary method when the objective is a black box you can only *evaluate*, not
differentiate — fitting a simulation's parameters to observed data, tuning a system with no
closed-form loss, optimizing a discrete layout (store shelf placement, cell tower positions), or
searching over program structure. If you have a differentiable loss and a framework that can
backprop through it, gradient-based optimization will almost always converge faster and more
reliably — don't reach for a swarm method by default just because it's flexible.

## The five algorithms and when each earns its keep

All operate on a population of vectors `x_i` in the search space, evaluated against an objective
function to minimize.

**Random Optimization (RO)** — the simplest possible baseline, no coordination between
particles:

```python
x_new = x + eta * x * rng.normal(size=x.shape) / 5   # eta ~ 0.1
if objective(x_new) < objective(x):
    x = x_new
```

Each particle perturbs itself and keeps the move only if it improves. No memory of other
particles, no adjustable structure beyond `eta`. Use it as a sanity-check baseline before reaching
for anything fancier — if a tuned PSO or DE can't beat plain RO on your problem, something else
(objective design, bounds, or the problem itself) is the bottleneck, not the algorithm choice.

**Jaya** ("victory," Sanskrit) — zero tunable parameters, moves every particle toward the
swarm's best and away from its worst simultaneously:

```python
x_i += r1 * (x_best - abs(x_i)) - r2 * (x_worst - abs(x_i))   # r1, r2 ~ Uniform[0,1) per component
```

Reach for Jaya first when you don't want to spend time tuning hyperparameters — it either works
well on a given problem shape or it doesn't, with nothing to adjust either way.

**Particle Swarm Optimization (PSO)** — canonical form tracks a velocity per particle:

```python
v = omega * v + c1 * r1 * (x_best_i - x) + c2 * r2 * (g_best - x)   # omega ~0.9 decaying, c1=c2~1.49
x = x + v
```

`x_best_i` is that particle's own best-ever position; `g_best` is the swarm's best-ever position.
`omega` (inertia) typically decreases over iterations, slowing particles as the search
presumably converges. **Bare-bones PSO** drops the velocity vector entirely and instead samples
new positions from a normal distribution centered between the particle's best and the swarm's
best:

```python
x_bar = (g_best + x_best_i) / 2
sigma = abs(g_best - x_best_i)
x_new = rng.normal(x_bar, sigma) if rng.random() < 0.5 else x_best_i   # per component
```

PSO family algorithms are a strong default for continuous, moderate-dimensional optimization
(curve fitting, parameter tuning) — fast convergence, moderate tuning burden.

**Differential Evolution (DE)** — builds a "donor" vector from three other random population
members, then crosses it with the current individual:

```python
# "DE/rand/1/bin": one random differential, binomial (per-gene) crossover
x1, x2, x3 = population[rng.choice(len(population), 3, replace=False)]
donor = x1 + F * (x2 - x3)                       # F ~ 0.8
offspring = np.where(rng.random(len(x_i)) < CR, donor, x_i)   # CR ~ 0.5
```

DE is not modeled on biology — it's a purely algorithmic recombination scheme — but it's
consistently one of the fastest-converging options for continuous optimization and a reasonable
second choice after PSO when PSO stalls.

**Genetic Algorithm (GA)** — closest to biological metaphor: crossover with a randomly chosen
mate from the top performers, plus a small chance of mutation, plus elitism (the current best
individual survives unchanged):

```python
if rng.random() < CR:                     # crossover probability, e.g. 0.8
    mate = top_performers[rng.integers(len(top_performers))]
    offspring = crossover(x_i, mate)       # e.g. uniform or single-point crossover
else:
    offspring = x_i
if rng.random() < mutation_rate:           # e.g. 0.05
    j = rng.integers(len(offspring))
    offspring[j] = rng.uniform(low[j], high[j])
```

GA tends to need an order of magnitude more iterations than PSO/DE to reach a comparable
solution on smooth numeric problems, but shines on problems that are naturally discrete/
combinatorial or high-dimensional enough to give evolution a large "genome" to work with —
scheduling, layout, and genetic-programming-style structure search.

## Decision guide

| Situation | Reach for |
|---|---|
| Fast baseline / sanity check | Random Optimization |
| Don't want to tune hyperparameters | Jaya |
| Continuous, moderate-dimensional, want fast convergence | PSO (canonical or bare-bones) |
| Continuous, PSO stalling or premature-converging | Differential Evolution |
| Discrete/combinatorial, or evolving structure (programs, layouts) | Genetic Algorithm |

## Objective function design is the actual bottleneck

Every algorithm above is only as good as the objective (fitness) function it's minimizing.
Two design mistakes cause the majority of "the swarm isn't converging" complaints:

- **Unbounded or wildly differently-scaled dimensions.** A search space where one dimension
  ranges `[0, 1]` and another ranges `[0, 1_000_000]` biases every distance- and
  perturbation-based update rule (RO's Gaussian step, PSO's velocity) toward the large-scale
  dimension. Normalize or rescale dimensions to comparable ranges before searching.
- **A flat or discontinuous objective landscape.** If most of the search space returns
  identical or wildly noisy objective values, no amount of algorithm tuning will produce
  meaningful gradient-like signal for the swarm to follow — redesign the objective (e.g., a
  smoother proxy, mean squared error instead of a hard pass/fail) before blaming the optimizer.

## Genetic programming: evolving code, not parameters

When you don't know the functional form to fit — only that some function `y = f(x)` should
approximate the data — evolve the *code* of `f` itself instead of a fixed set of parameters.
Represent candidate expressions in **postfix (reverse Polish) notation** so that a stack machine
can both evaluate them and treat their token sequence as a position in a search space any of the
algorithms above can search:

```python
def eval_postfix(tokens, x):
    """tokens: e.g. ['x', 'x', '*', '2', '+']  =>  x*x + 2"""
    stack = []
    for t in tokens:
        if t == 'x':
            stack.append(x)
        elif isinstance(t, (int, float)):
            stack.append(t)
        else:
            b, a = stack.pop(), stack.pop()
            stack.append({'+': a+b, '-': a-b, '*': a*b,
                           '/': a/b if b else 1.0}[t])
    return stack[0]
```

A token sequence of fixed length becomes a fixed-length vector any swarm/GA algorithm can
mutate and recombine directly — no need for tree-crossover machinery. Fitness is typically mean
squared error between `eval_postfix(tokens, x_i)` and the observed `y_i` across the dataset.
This is the same technique underlying symbolic regression tools; expect it to need far more
iterations than plain curve fitting (you're searching over program structure, a vastly larger
space than a handful of continuous parameters), and expect GA (not PSO/DE) to be the natural
fit given the discrete, structural nature of the search.

## Common pitfalls

- **Premature convergence** — the whole swarm collapses onto one local optimum early, especially
  with PSO's inertia decaying too fast or DE's `F`/`CR` set too conservatively. Track diversity
  (e.g., mean pairwise distance between particles) alongside best-fitness; if diversity collapses
  while fitness plateaus far from a known-good value, increase mutation/exploration parameters.
- **Comparing algorithms on iteration count alone.** GA needing "10x more iterations" than DE to
  reach a similar result may still be cheaper in wall-clock time if each GA iteration is far
  simpler — measure objective-function evaluations and wall-clock time, not just loop count.
- **Reusing one fixed random seed across a "robustness" test.** Swarm methods are inherently
  stochastic; a single successful run proves nothing about typical behavior — run multiple
  seeds and report a distribution (mean ± std or a box plot) of final objective values.
- **Letting bounds silently clip in a way that hides a bug.** If Bounds/Validate logic forces
  values back into range every iteration, verify particles aren't spending most of their time
  pinned at a boundary — that usually means the true optimum lies outside the assumed range.
- **Not normalizing dimensions with wildly different scales**, which biases distance- and
  perturbation-based update rules toward whichever dimension has the largest numeric range.

## Checklist

- [ ] Confirmed there's no usable gradient before reaching for a population-based method —
      gradient-based optimization is faster and more reliable when it applies
- [ ] Chose an algorithm from the decision guide based on problem shape (continuous vs.
      discrete/structural), not by default habit
- [ ] Search space dimensions are normalized/rescaled to comparable ranges
- [ ] Objective function is smooth/informative enough to distinguish "somewhat better" from
      "much better," not just pass/fail
- [ ] Ran multiple random seeds and reported a distribution of outcomes, not a single lucky run
- [ ] Tracked population diversity alongside best-fitness to catch premature convergence
- [ ] For genetic programming / structure search, used a fixed-length representation (e.g.
      postfix token sequence) so existing swarm/GA machinery applies without custom
      tree-crossover code
