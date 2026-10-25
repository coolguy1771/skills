---
name: dynamic-programming-problem-recognition
description: Recognize optimization/counting problems that need memoization or dynamic programming (DP), characterize their optimal substructure, and choose between a top-down memoized recursion and a bottom-up DP table. Use this when the user has a brute-force recursive solution that's exponential/too slow, is asking "how do I speed up this recursion", is solving a problem about minimizing/maximizing/counting ways to reach a target (coin change, knapsack-style, "fewest X to make Y", "number of ways to..."), mentions "overlapping subproblems", or is deciding whether a problem needs memoization vs plain recursion vs a graph algorithm. Complements recursive-decomposition-and-tree-algorithms (independent subproblems, no memo needed) and graph-search-and-shortest-paths (use when the subproblem dependency graph has a cycle).
---

# Dynamic Programming: Problem Recognition and Method

DP is not a grab-bag of tricks to memorize per problem — it's a four-step method that applies
whenever a problem has **optimal substructure**: an optimal solution to the whole problem is
built from optimal solutions to smaller subproblems of the *same kind*. The four steps, in
order, are the actual work; skipping straight to "write a DP table" without doing step 1
honestly is why DP problems feel like guesswork.

## Recognizing the pattern

Signs a problem is a DP candidate:

- It asks to **minimize**, **maximize**, or **count the number of ways** to achieve something
  (fewest coins/items to reach a total, longest/shortest sequence satisfying a property, number
  of distinct ways to reach a target).
- A brute-force recursive solution is *correct* but exponential, because it tries every
  combination of choices.
- You can express "the best answer for the full problem" in terms of "the best answer for a
  strictly smaller version of the same problem," for every way the last choice could have gone.

The critical distinguishing test against plain recursion (see
`recursive-decomposition-and-tree-algorithms`): **do different sequences of choices ever lead
to the identical smaller subproblem?** If yes, you have *overlapping subproblems*, and plain
recursion will recompute the same answer exponentially many times — that recomputation is
exactly what memoization/DP eliminates. If every subproblem is reached exactly one way (as in
tree recursion), you don't need DP at all; recursion alone is already efficient.

**The cycle test** for choosing DP vs. a graph algorithm: draw an arrow from each subproblem
to the smaller subproblems it depends on. If that dependency graph is guaranteed acyclic (you
always recurse toward strictly "smaller" — fewer minutes remaining, fewer items left, a smaller
index) — DP applies. The moment subproblems can depend on each other in a cycle (a state can
be reached and re-reached without any measure of it ever strictly decreasing — as when steps
can move a quantity up or down in ways that loop back to a previously seen state), it's a graph
problem instead — see `graph-search-and-shortest-paths`.

## The four-step method

### Step 1 — Structure of optimal solutions

Before writing any code, answer: **if I knew the very last choice made in an optimal solution,
what smaller subproblem would remain, and would an optimal solution to that smaller
subproblem have to be part of the optimal solution to the whole thing?**

This is the step people skip, and it's the one that actually requires thinking about the specific
problem. Concretely:

- Identify what "the last choice" could be (which coin/item was used last, which edge was
  taken last, etc.) and enumerate the possibilities.
- For each possibility, state the smaller subproblem that remains.
- Argue (even informally) that an optimal solution to the full problem must contain an
  *optimal* — not just any — solution to that remaining subproblem. (If it didn't, you could
  swap in the better sub-solution and get something even better, contradicting optimality —
  this is the standard "optimal substructure" argument, and it's worth actually making it, not
  assuming it.)
- Make sure the subproblems are **genuinely smaller** in some measure that strictly decreases
  (remaining budget, remaining items, a smaller index) — otherwise there's no base case to
  bottom out at, and no valid recursion.

If you can't find such a decomposition, the problem may not have optimal substructure, and DP
is the wrong tool — reconsider whether it's actually a graph, greedy, or search problem.

### Step 2 — Recursive solution

Step 1 hands you the recursive solution directly: to solve the full problem, try each candidate
for "the last choice," recursively solve the resulting smaller subproblem optimally, and take the
best (min/max/sum, depending on whether you're optimizing or counting) over all candidates.

```python
def solve(remaining):
    if remaining == 0:
        return 0                       # base case: no more remaining, no more choices needed
    if remaining < 0:
        return float('inf')            # sentinel: this path is invalid
    best = float('inf')
    for choice in candidates:          # try every possible "last choice"
        best = min(best, 1 + solve(remaining - choice))
    return best
```

This is correct but, if subproblems overlap, exponentially slow — every distinct sequence of
choices that lands on the same `remaining` re-derives the same answer from scratch.

### Step 3 — Memoization

Store each subproblem's answer the first time it's computed; look it up instead of recomputing
it every subsequent time it's needed. This alone typically takes an exponential recursion down
to polynomial time, because now every distinct subproblem is solved exactly once:

```python
def solve(remaining, memo={}):
    if remaining == 0:
        return 0
    if remaining < 0:
        return float('inf')
    if remaining in memo:
        return memo[remaining]
    best = float('inf')
    for choice in candidates:
        best = min(best, 1 + solve(remaining - choice, memo))
    memo[remaining] = best
    return best
```

This is the "top-down" form: it still uses recursion, but never solves the same subproblem twice.

### Step 4 — Dynamic programming (bottom-up, optional)

Once memoization works, you can eliminate the recursion entirely by observing that subproblems
can be solved in a fixed order — smallest first — and filled into a table with a loop instead of
recursive calls:

```python
def solve(total, candidates):
    dp = [float('inf')] * (total + 1)
    dp[0] = 0
    for remaining in range(1, total + 1):
        for choice in candidates:
            if choice <= remaining:
                dp[remaining] = min(dp[remaining], 1 + dp[remaining - choice])
    return dp[total]
```

**Memoization vs. bottom-up DP — pick based on this, not habit:**

- Memoization solves subproblems **on demand**, so if large parts of the subproblem space are
  never actually reached (e.g., only even remainders ever occur given the specific choices
  available), it can end up doing meaningfully less work than filling in the entire table.
- Bottom-up DP has no recursion overhead (no function-call stack) and is often simpler to
  reason about for correctness once you're comfortable with the ordering — but it pays for
  every cell in the table even if some are never on a path to the answer.
- Multi-dimensional DP tables (subproblems parameterized by two or more quantities) are where
  bottom-up's bookkeeping — base cases and iteration bounds in every dimension — gets error
  prone; if that's fiddly, prefer memoization, where each dimension is just another argument
  to the function and the base cases are ordinary early-returns.
- When in doubt, implement memoization first (it's usually less bug-prone to get right), then
  convert to bottom-up only if the recursion overhead is a proven bottleneck.

## Extending to more subproblem dimensions

Not every DP problem's subproblem is "a smaller number." Two extensions to look for:

- **Multiple parameters**: e.g., "cheapest way to reach exactly step `i` having used exactly `k`
  jumps" — the subproblem is indexed by a pair `(i, k)`, and the memo/table has a dimension per
  parameter.
- **"Exactly" vs. "at most" subproblems**: sometimes the natural subproblem ("ways to make
  exactly this total") is awkward to combine, and reformulating it as a different but related
  subproblem (e.g., adding an auxiliary dimension distinguishing "exactly" from "at most") makes
  the recursion cleaner. If a formulation is producing 4+ special cases in the recursive step,
  it's worth trying a reformulation with one more dimension before pushing through the mess.

## Complexity

Total work is (number of distinct subproblems) × (work per subproblem, e.g., number of choices
tried). A single-integer subproblem space with `c` choices per subproblem and target `n` is
O(n·c). A pair-indexed subproblem space is O(n·m·c). This is why identifying the *exact* set of
distinct subproblems in Step 1 matters for complexity analysis, not just correctness — it's
usually the product of the ranges of every subproblem dimension.

## Common pitfalls

- **Skipping Step 1 and guessing a recurrence** — leads to recurrences that look plausible but
  are wrong on some input, because the optimal-substructure argument was never actually made.
- **Subproblems that aren't strictly smaller** — a "recursion" that can call itself with the same
  or a larger parameter never bottoms out; if you can't order all subproblems from smallest to
  largest, something is wrong with the decomposition (or it's actually a graph problem — see
  the cycle test above).
- **Wrong sentinel values** — using `0` as a placeholder for "unreachable" in a `min` DP silently
  makes unreachable states look like the best possible answer; use `+infinity`/`-infinity`
  (or a distinct "unreached" marker) consistently and check for it.
- **Off-by-one in loop bounds for bottom-up tables**, especially multi-dimensional ones —
  getting the base cases and iteration ranges wrong is the single biggest reason bottom-up DP
  is harder to get right than memoization on the first attempt.
- **Assuming DP is "formulaic"** — recognizing optimal substructure and getting the recursion
  right for a *new* problem shape is a genuine reasoning step every time; treat each problem's
  Step 1 as real work, not a template to fill in.

## Checklist

- [ ] Stated explicitly what "the last choice" in an optimal solution could be, and enumerated
      the possibilities
- [ ] Argued why an optimal sub-solution to the remaining subproblem must be part of the
      overall optimal solution (optimal substructure)
- [ ] Confirmed subproblems are strictly smaller by some measure that bottoms out at a base case
- [ ] Confirmed the dependency graph between subproblems has no cycle (otherwise: graph
      algorithm, not DP)
- [ ] Verified subproblems actually overlap (otherwise: plain recursion is already sufficient)
- [ ] Wrote the brute-force recursive solution first and validated it against small/known cases
      before adding memoization
- [ ] Chose sentinel values (`+inf`/`-inf`/unreached marker) appropriate to min/max/count and
      applied them consistently
- [ ] Chose memoization vs. bottom-up deliberately (sparse reachable subproblem space and/or
      multi-dimensional state → lean memoization; dense space and simple state → either works)
