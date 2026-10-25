---
name: unfamiliar-problem-decomposition-strategy
description: Systematic heuristics for attacking a programming problem you don't already know how to solve — when there's no obvious algorithm to reach for, you're staring at a blank editor, or you feel stuck/frustrated and don't know where to start. Use this when the user says things like "I don't know how to approach this," "where do I even start," "I'm stuck," "how do I break this problem down," "this problem seems too big/vague," or asks for help planning a solution to an unfamiliar or loosely-specified problem before any code gets written. This is about the *meta*-process of finding a plan (restating, dividing, reducing, analogy, controlled experimentation) — not about a specific algorithm or data structure; for those, prefer the algorithm/data-structure-specific skills (e.g. dynamic-programming-problem-recognition, recursive-decomposition-and-tree-algorithms, hash-table-lookup-patterns) once the shape of the problem is already known.
---

# Unfamiliar Problem Decomposition Strategy

Most "I don't know how to solve this" moments aren't a knowledge gap about a specific
algorithm — they're a missing *plan*. This skill is a small set of general-purpose moves
that turn a vague, intimidating problem into one or more concrete, already-known problems.
Apply them in roughly this order, but circle back whenever one unblocks new information.

## The core rule: always have a plan

Never let "just try things and see what happens" be the whole strategy — that's guessing,
not problem solving. Before writing code (or even before you know the eventual algorithm),
write down a plan with intermediate milestones, even tentative ones: "1) restate the problem
in my own words, 2) figure out what subpart I already know how to do, 3) prototype that part,
4) figure out what's left." A plan gives you checkable progress and a place to update when
reality changes it — an outline surviving contact with the problem is a bonus, not the point.

If a step of the plan turns out to be wrong, that's not a failure of planning, it's what
planning is for: you now have a *new*, more informed plan instead of nothing.

## The techniques

Apply these to the problem itself, not to code you've already started writing — they're
about finding the plan, before or during design.

### 1. Restate the problem

Rewrite the problem in your own words, and in more than one way. A problem that looks hard
in its original phrasing can look trivial in another. Two payoffs beyond insight:
- It surfaces whether the *stated* goal is really the goal. (Classic trap: someone asks for
  a baby to stay inside a play pen, but the actual goal is "let me get something done in
  peace" — a different solution, letting the baby play on the floor while the adult works
  inside the pen, satisfies the real goal better.) Ask "what am I actually trying to achieve"
  before committing to the literal ask.
- If someone else assigned the problem, restating it back to them cheaply confirms you
  understood it correctly before you invest hours solving the wrong thing.

Restatement is also the prerequisite step for dividing or reducing (below) — you usually
can't see the natural cut points in a problem until you've said it in your own terms.

### 2. Divide the problem

Look for a way to split the problem into phases or independent pieces. Splitting isn't just
"half the size, half the difficulty" — it's frequently *far* cheaper than that, because the
difficulty of many problems grows faster than linearly with size (sorting 4 groups of 25 items
is much less than a quarter of the work of sorting 100 items in one pass). The same applies to
control-flow complexity: a function with an `if` nested in a `while` nested in a `for` is much
harder to write and read than three separate, sequential loops/branches, even though it's "the
same amount" of logic.

Some divisions are hidden — e.g. a puzzle whose state can be represented as several independent
sub-states that can each be minimized separately. If you can't see a division, try reducing the
problem first; the division sometimes appears out of the reduced version.

### 3. Start with what you know

Once divided (or even if not), identify the piece(s) you already know how to code, and build
those first — don't wait until you have a plan for the *entire* problem before writing anything.
This does two things: it builds working, checkable progress (morale and momentum matter — a
program that does nothing useful until it's 90% done is a recipe for frustration), and a partial
working solution often reveals how the remaining pieces should fit together. Investigate the
problem fully using the skills you already have before reaching for something new.

### 4. Reduce the problem

When you can't solve the problem as given, temporarily change its constraints — add ones that
make it easier, or drop ones you don't yet know how to satisfy — until you land on a problem you
*do* know how to solve. Example: "find the two closest points among N points in 3D space" is
hard if you don't know 3D distance yet; reduce to 2D, or to points on a single line (find the
two closest numbers in a list), or reduce N down to exactly 3 points (just compare all pairs).
Each reduction isolates a different piece of difficulty. You'll eventually need to recombine
them, but each reduced version makes real progress and tells you exactly which sub-skill is
still missing.

This is also the sharpest tool for asking someone else for help: instead of "here's my program,
it doesn't work, why not," reduce until you can say "I can do X and Y, but I don't see how to
combine them to get Z" — a precise, answerable question.

### 5. Look for analogies

Ask: does this resemble a problem I've already solved, even partially? An analogy can be exact
(this new puzzle is really the same puzzle in disguise) or partial (unrelated problem, but both
need the same trick for extra-precision arithmetic). Analogies are the single biggest speed
multiplier in problem solving, and the only way to build a stockpile of them is to have actually
solved problems yourself, from scratch, in the past.

This is why copying and adapting someone else's code as a shortcut is a trap even when it
"works": if you didn't produce the solution yourself, you don't have it as a future analogy, and
you'll need to borrow again next time. Every problem you genuinely solve is doubly valuable —
once as this solution, and again as future pattern-matching material.

### 6. Experiment (not guess)

Guessing is typing code and hoping. Experimenting is: form a specific hypothesis about what a
piece of code, a library, or a language feature will do, run a small isolated test to check it,
and use the result — whichever way it comes out — to narrow down the real problem.

- Unfamiliar API/library: before wiring it into the real solution, write a tiny throwaway
  program that pokes at exactly the behavior you're unsure of (what happens on delete, what a
  boundary call returns). Keep it around as a reference for next time.
- Debugging-flavored experiment: if output is wrong in a specific way (e.g., reversed), don't
  just stare at the code — deliberately modify a copy to force the suspected behavior (e.g. run
  the loop backwards on purpose) and see whether the output changes as predicted. Either outcome
  teaches you something concrete about where the real bug is.

### 7. Don't get frustrated — and don't treat this as unactionable advice

Frustration is not an uncontrollable reflex; it's a choice to give yourself an excuse to keep
failing. Two concrete countermeasures, not willpower:
- Fall back on the plan. If you have a plan with intermediate steps and you're executing it,
  you're making progress even before the "real" code compiles — believe that. If you've run out
  of plan and still have nothing, that's the signal to make a *new* plan (usually by reducing or
  dividing further), not to grind.
- Keep more than one thread available. If a divided sub-problem has you stuck, switch to a
  different piece rather than staring at the blocker. If there's truly only one problem, step
  fully away — a walk, anything that isn't "stare harder" — rather than continuing to grind once
  frustration has set in; frustrated work is measurably worse work.

## Worked shape (how these compose)

A real session typically interleaves techniques rather than running through them once in order:
restate → attempt to divide → for any piece you can't divide further, reduce it until it matches
something you know or resembles something you've solved → prototype the known/reduced pieces
first → use what you learn from experimenting on the unclear parts to refine the plan → recombine.
Treat the list as a toolbox to revisit, not a waterfall.

## Common pitfalls

- **Skipping straight to code** because restating/dividing "isn't real progress." It is — put it
  on the plan explicitly as step 1 so it counts.
- **Reducing and forgetting to recombine** — a reduced problem is a stepping stone, not the
  destination; track which constraints you dropped and revisit them.
- **Treating an unfamiliar library/problem shape as something to guess at** instead of writing a
  small controlled experiment to resolve the specific uncertainty.
- **Borrowing someone else's code to "get past" a stuck point** instead of reducing the problem
  to something you can solve yourself — it removes the analogy you'd otherwise have banked for
  next time, and you can't reliably debug or extend code you didn't derive yourself.
- **No plan beyond "solve the whole thing"** — leads to no sense of progress until the very end,
  which is exactly what makes hard problems feel unbearable.

## Checklist

- [ ] Restated the problem in your own words (and confirmed understanding with whoever assigned
      it, if applicable)
- [ ] Looked for a way to divide the problem into independent or sequential pieces
- [ ] Identified which piece(s) you can already solve and started there
- [ ] For any piece you can't solve, found a reduction (fewer dimensions, smaller N, fewer
      constraints) that you *can* solve, and noted what you dropped
- [ ] Checked for analogies to problems you've solved before, exact or partial
- [ ] Replaced any guessing with a specific, falsifiable experiment
- [ ] Wrote down a plan with checkable intermediate milestones before starting to code in earnest
