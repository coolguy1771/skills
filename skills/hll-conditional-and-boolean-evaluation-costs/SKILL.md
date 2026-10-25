---
name: hll-conditional-and-boolean-evaluation-costs
description: Reason about the compiled cost of if/switch statements and Boolean expressions in high-level code — short-circuit vs. complete evaluation, jump tables vs. chained comparisons for switch/case, and the hidden cost of calling a function inside a loop or if condition. Use this when the user asks "why is my if-chain slow with many branches," "when does switch beat if-else," "does my language short-circuit && and ||," "why did moving a function call out of a while condition speed things up," "is a jump table always used for switch," "how do I force short-circuit or complete Boolean evaluation," or is reviewing a hot conditional/dispatch path, a compound Boolean expression with side effects, or a loop whose termination test calls a function (e.g. `while (i < strlen(s))`). Distinct from arm64-control-flow-and-loop-optimization (writing the ARM64 assembly for control flow) and compiler-pipeline-lexing-parsing-optimization (general optimizer transformations) — this is about choosing and ordering high-level conditionals with their compiled cost in mind, in any language.
---

# HLL Conditional and Boolean Evaluation Costs

The same `if`/`switch`/loop-condition source code compiles to meaningfully different
instruction counts depending on how the compiler evaluates Boolean expressions and how
it implements multi-way branches. Two pieces of code that "do the same thing" at the
source level can differ by 2-10x at the machine level based entirely on these choices —
and some of the choices change *observable behavior* (not just speed), which is the more
dangerous trap.

## Short-circuit vs. complete Boolean evaluation

- **Short-circuit evaluation**: `A && B` skips evaluating `B` entirely if `A` is false
  (and `A || B` skips `B` if `A` is true), because the result is already determined.
  C, C++, C#, Swift, and Java *guarantee* short-circuit evaluation for `&&`/`||`. Ada and
  HLA let you choose. Pascal leaves it to the implementation — the same compiler may use
  either scheme for different occurrences of the same expression.
- **Complete evaluation**: every subexpression is always evaluated, then the logical
  operators combine the already-computed results.
- **Short-circuit is a correctness mechanism, not just an optimization**, whenever a
  later subexpression is only safe to evaluate because an earlier one gated it:

  ```c
  if (ptr != NULL && *ptr != '\0') { /* process *ptr */ }
  ```

  Under complete evaluation this dereferences `ptr` even when it's `NULL`. Never assume
  a guard-then-use pattern like this is safe unless you know your language mandates
  short-circuit evaluation for the operator you're using.
- **Short-circuiting also means skipped side effects.** A subexpression's function calls,
  assignments, or `++`/`--` don't happen if short-circuiting skips it. This cuts both
  ways — some code (like a cursor-advance idiom `*++p && *++p && ...`) *depends on* the
  early exit; other code silently breaks if a "guaranteed" side effect stops firing once
  an earlier operand decides the outcome. When side effects must always run regardless of
  outcome, force complete evaluation explicitly rather than relying on the operator.

**Forcing complete evaluation** in a short-circuiting language: evaluate each
subexpression into a temporary first, then combine the temporaries so the operator has
nothing left to skip:

```c
// Complex expression: if ((a < f(x)) && (b != g(y)) || predicate(a + b))
int temp1 = a < f(x);
int temp2 = b != g(y);
int temp3 = predicate(a + b);
if ((temp1 && temp2) || temp3) { ... }   // f(), g(), predicate() always run
```

**Forcing short-circuit evaluation** in a language that only does complete evaluation:
gate each later subexpression's computation behind an `if` on the running result, instead
of writing one flat Boolean expression.

## The loop-condition function-call trap

The single most common real-world instance of this is a function call sitting directly in
a loop's test — evaluated on *every* iteration even though the compiler cannot always
prove it's safe to hoist or cache:

```c
// Recomputes strlen(s) — an O(n) scan — on every single iteration.
// The whole loop is silently O(n^2).
while (i < strlen(s)) { ... i++; }

// Fix: compute once, outside the loop, since the loop body doesn't
// change the string's length.
size_t n = strlen(s);
while (i < n) { ... i++; }
```

More generally, any non-trivial function call embedded in an `if`/`while`/`for` test pays
full call/return overhead (parameter setup, `call`, prologue, epilogue, `ret`) every time
the condition is checked — this is frequently *more expensive than the comparison it
gates*. If the compiler can prove the function is pure and its inputs unchanged in the
loop, it may hoist or inline it (see `function-call-overhead-and-parameter-passing` for
when inlining actually happens) — but don't rely on that proof going through for anything
that touches memory the optimizer can't fully track (a pointer parameter, a global, a
call across translation units). When in doubt, hoist the call yourself and re-derive the
cached value only where the loop body actually invalidates it.

## switch/case: jump table vs. chained comparisons

A `switch` is not just syntactic sugar for an `if..elseif` chain — semantically it's
*more restrictive* (case labels must be compile-time constants, and you can only test one
expression), and that restriction is exactly what lets the compiler use a fixed-cost
dispatch instead of a linear scan:

```
Chained comparison (if..elseif): O(k) — up to k compares+branches for k cases,
                                  average k/2 for uniformly likely cases.

Jump table:                      O(1) — one bounds check, one indexed load,
                                  one indirect jump, regardless of case count.
```

Conceptually, the compiler builds a table of code addresses indexed by the switch value
and jumps through it:

```c
// switch (i) { case 0: A(); break; case 1: B(); break; case 2: C(); break; case 3: D(); break; }

// Compiles to roughly:
//   if ((unsigned)i > 3) goto done;      // range check
//   goto *jumpTable[i];                  // indexed indirect jump
// jumpTable = { &&case0, &&case1, &&case2, &&case3 };
```

Key facts that determine which strategy a real compiler picks, and that you should use
when reasoning about a hot `switch`:

- **Break-even point is roughly 3-4 cases.** A jump table needs a range check plus an
  indexed indirect jump (an indirect branch typically costs more than a direct
  conditional branch) — for very few cases, a plain compare chain wins outright.
- **Dense case values are cheap; sparse ones are expensive in *memory*, not time.**
  `case 0, 1, 5, 8:` needs a 9-entry table (with dead entries pointing at the
  default/break label) to stay O(1)-indexable. `case 0, 1, 10, 100, 1000, 10000:` would
  need a 10,001-entry table — technically still O(1) time, but the compiler will
  reasonably refuse and fall back to a comparison chain (or a small binary-search table)
  once the value range is wildly larger than the case count. Don't assume "switch = jump
  table" for sparse, widely spaced case values.
  - Watch table *size* effects on cache behavior separately — a huge jump table that
    doesn't fit in cache can make even O(1) dispatch slower via cold-line misses; that's
    a memory-hierarchy concern (see `memory-hierarchy-and-cache-optimization`), not a
    dispatch-algorithm one.
- **A `switch` cannot express what an `if..elseif` chain can.** Non-constant case labels,
  or testing different expressions per branch, are simply illegal in a `switch` — that
  restriction is the price of admission for O(1) dispatch, not an arbitrary language
  limitation.
- Check your actual compiler's output (`-S`, a disassembler, or a Compiler Explorer-style
  tool) before assuming which strategy it picked — some compilers use jump tables far
  more conservatively than others, and "the switch clause count is high" is a hint, not a
  guarantee.

## Common pitfalls

- **Assuming `&&`/`||` short-circuit in a language that doesn't guarantee it** (Pascal
  and similar) and writing a null-check-then-dereference pattern that only "happens to
  work" with your current compiler.
- **Leaving a pure, loop-invariant function call inside a loop's test condition** —
  the classic `while (i < strlen(s))` — and blaming an O(n) algorithm for O(n²) real
  behavior.
- **Expecting a `switch` with widely spaced, sparse case values to compile to a huge
  fast jump table** — the compiler will often (reasonably) fall back to comparisons, and
  even when it does build the table, its size can hurt cache behavior more than a compare
  chain would have.
- **Converting a small (2-3 case) switch from an if-chain "for speed"** — below the
  break-even point, the extra range check and indirect jump can make it slower, not
  faster.
- **Forcing complete Boolean evaluation "for clarity" without checking for side
    effects** — if any subexpression has a side effect the algorithm depends on
  running unconditionally (or NOT running, per short-circuit semantics), converting
  the evaluation strategy changes program behavior, not just speed.

## Checklist

- [ ] Confirmed whether the language guarantees short-circuit evaluation for `&&`/`||`
      before writing a guard-then-use conditional (null check then dereference, bounds
      check then index, etc.)
- [ ] Checked every loop/if condition for a function call whose result doesn't change
      across iterations, and hoisted it if the compiler can't be trusted to
- [ ] Verified no subexpression in a Boolean expression has a side effect the algorithm
      silently depends on (or depends on *not* running) under the language's evaluation
      order
- [ ] For a hot `switch` with few (≤3-4) cases, considered whether an `if..elseif`
      chain is actually cheaper than a jump table's range-check-plus-indirect-jump
- [ ] For a hot `switch` with sparse, widely spaced case values, checked the actual
      compiler output rather than assuming a large jump table was built
- [ ] Ruled out a giant jump table's cache footprint as the cause of unexpected slowness
      in an otherwise O(1) dispatch
