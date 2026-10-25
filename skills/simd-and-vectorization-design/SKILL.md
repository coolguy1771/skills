---
name: simd-and-vectorization-design
description: Reason about SIMD (single instruction, multiple data) and vectorization when writing or reviewing numeric, image/audio, or bulk data-processing loops for performance, or when the user asks "why isn't the compiler auto-vectorizing this loop", "how do I write SIMD-friendly code", "should I use intrinsics/AVX/SSE/NEON here", "why is this loop with a branch/dependency not vectorizing", or when the workload involves applying the same operation across many independent numeric elements (per-pixel/per-sample/per-vertex processing, elementwise math on arrays, dot products, bulk numeric transforms). Also load this when comparing data layouts (array-of-structs vs struct-of-arrays) specifically for vectorizability, or when deciding whether a workload belongs on CPU SIMD, a GPU, or neither.
---

# SIMD and Vectorization Design

SIMD (single instruction, multiple data) lets one instruction apply the same operation to several
data elements packed into one wide register at once (e.g., four 32-bit values in a 128-bit
register, or eight in a 256-bit register), rather than issuing one instruction per element. This
is a distinct axis of parallelism from the instruction-level parallelism covered in
`cpu-pipeline-hazards-and-ilp` — SIMD parallelizes *within* a single instruction's data, not
across nearby instructions. The payoff can be a 4x-16x throughput increase on the right workload,
but only if the loop's data and control flow are shaped so the hardware (or the compiler's
auto-vectorizer) can actually pack it.

## Recognizing a SIMD-shaped workload

The signature is: **the same operation, applied independently and uniformly to many data
elements, with no data-dependent control flow between elements.** Concretely:

- Elementwise arithmetic over arrays (`for i: c[i] = a[i] + b[i]`)
- Per-pixel/per-sample transforms (brightness/gamma adjustment, audio gain, format conversion)
- Reductions with an associative operator (sum, dot product, min/max) — vectorizable with care
  (see below)
- Small fixed-size vector/matrix math (3D transforms, quaternions) — this is precisely why 4-wide
  SIMD lanes and 4D homogeneous coordinates are such a natural fit for graphics: a single SIMD
  instruction is exactly one vector operation.

The signature that *breaks* SIMD-shaping is any point where different lanes need to take
different actions based on their own data — a branch inside the loop whose outcome varies
per-element, or a loop-carried dependency where iteration `i` needs iteration `i-1`'s *actual
computed result* (not just its index) before it can proceed.

## Why data layout decides whether vectorization is possible at all

SIMD instructions load contiguous, uniformly-typed lanes directly into a wide register. This
means the **struct-of-arrays (SoA)** layout from `memory-hierarchy-and-cache-optimization` isn't
just a cache-friendliness choice here — it's usually a prerequisite for vectorization:

```c
// AoS: x, y, z are interleaved in memory. A SIMD load here would pull in
// x0,y0,z0,x1 into one register — not four x-values — so it can't be used directly
// for a "multiply every x by 2" operation without an expensive gather/shuffle first.
struct Point { float x, y, z; };
Point pts[N];
for (int i = 0; i < N; i++) pts[i].x *= 2.0f;

// SoA: all x values are contiguous, so four (or eight) of them load directly into
// one SIMD register and get multiplied in a single instruction.
float xs[N], ys[N], zs[N];
for (int i = 0; i < N; i++) xs[i] *= 2.0f;   // trivially auto-vectorizable
```

If a loop won't auto-vectorize and profiling says it should matter, check data layout before
reaching for manual intrinsics — often a layout change alone lets the compiler's auto-vectorizer
do the rest for free.

## What actually blocks auto-vectorization (and how to unblock it)

Modern compilers vectorize simple loops automatically, but commonly bail out on:

- **Aliasing uncertainty**: if the compiler can't prove two pointer arguments don't overlap, it
  can't safely reorder/pack their accesses. Use `restrict` (C/C++) or language-level guarantees
  (e.g., slices vs raw pointers) to tell the compiler the arrays are independent.
- **Data-dependent branches inside the loop body**: a per-element `if` that can go either way
  defeats packing multiple elements into lockstep execution (this is exactly why unpredictable
  branches matter here too — see `branch-prediction-and-speculative-execution`). Where possible,
  rewrite as a branchless/masked computation (compute both outcomes, select with a mask) so every
  lane does the same operation.
- **Loop-carried true dependencies**: `a[i] = a[i-1] + x[i]` can't vectorize as written, because
  each output genuinely needs the previous output. Reductions look like this but usually aren't a
  real blocker — see below.
- **Non-contiguous/strided access**: gathering scattered elements into a SIMD register is far
  more expensive than a contiguous load, sometimes negating the benefit entirely. This is the
  same locality argument as `memory-hierarchy-and-cache-optimization`, applied specifically to
  whether a *vector load* is cheap.
- **Mixed or narrow/odd-sized types** that don't divide evenly into a register's lane width can
  leave a "remainder" that must be handled scalar-style; usually a minor tail cost, not a reason
  to avoid vectorizing the main body.

## Reductions are usually vectorizable despite looking loop-carried

A naive reduction looks sequentially dependent, but if the operator is associative (sum, product,
min, max, bitwise and/or), it can be restructured into independent partial reductions — the same
"multiple accumulators" trick from `cpu-pipeline-hazards-and-ilp`, but done in SIMD lanes instead
of scalar registers:

```c
// Looks loop-carried, but + is associative: partial-sum in SIMD-width-sized lanes,
// then do one final horizontal combine of the lanes at the end.
float sum = 0;
for (int i = 0; i < n; i++) sum += a[i];
```

Compilers can often auto-vectorize this specific shape for floating-point sums *only* if allowed
to reassociate (floating-point addition isn't strictly associative due to rounding) — this is why
strict IEEE-754 compliance flags can silently disable an otherwise-available vectorization, and
why "fast-math"-style compiler flags exist. Decide deliberately whether bit-for-bit reproducible
floating-point results or raw throughput matters more for the specific computation before
enabling that trade-off project-wide.

## When to use explicit intrinsics vs. trust the compiler

Reach for hand-written SIMD intrinsics (SSE/AVX on x86, NEON on ARM, etc.) only after confirming,
via profiling and by inspecting generated assembly/vectorization reports, that:

1. The loop is genuinely hot enough to matter, and
2. The compiler's auto-vectorizer is failing to vectorize it (or vectorizing it suboptimally),
   and
3. A layout/aliasing/branch fix (above) doesn't unblock the auto-vectorizer instead.

Hand intrinsics are harder to read, port, and maintain (they're tied to a specific instruction
set generation), so they're a last resort for a confirmed hot path, not a default style. Where
available, portable vector abstractions (compiler-provided vector types, or library-level
SIMD wrappers) are a good middle ground between plain scalar code and raw intrinsics.

## SIMD vs. GPU vs. plain scalar: choosing the right tool

- **Plain scalar** is fine for anything not hot, or for control-flow-heavy logic where different
  elements genuinely need different code paths.
- **CPU SIMD** wins for hot, elementwise, moderately sized numeric workloads that stay largely in
  cache and don't need massive parallelism — the packing/unpacking and kernel-launch overhead of
  a GPU wouldn't pay for itself, and CPU SIMD keeps the data local to code that also needs to do
  other (branchy, sequential) work nearby.
- **GPU** wins when the same operation must be applied to *very* large numbers of independent
  elements (thousands+) and the data can be transferred/kept resident on the device — the
  overhead of moving data to/from the GPU and launching kernels only pays off at that scale. See
  `parallel-architecture-and-scaling-limits` for the broader tradeoff between SIMD, multicore,
  and GPU/cluster-style parallelism.

## Common pitfalls

- **Writing AoS-style code for a numeric hot path and expecting auto-vectorization.** Check
  layout first — see the worked example above.
- **Leaving a data-dependent branch inside a "vectorizable-looking" loop** and being surprised the
  compiler didn't vectorize it — rewrite as branchless/masked logic first.
- **Assuming a reduction can't vectorize** because it "looks" loop-carried — check whether the
  operator is associative and whether reassociation is acceptable for the required numeric
  precision.
- **Jumping straight to hand-written intrinsics** without first checking a vectorization report
  or trying a layout/aliasing fix — intrinsics lock in a specific instruction-set width/generation
  and are a maintenance cost that's often unnecessary.
- **Vectorizing a workload that's actually too small or too control-flow-heavy to benefit**,
  paying the packing/masking overhead for no real throughput gain.
- **Ignoring the tail/remainder** when the data length isn't a multiple of the SIMD width —
  ensure correctness of the scalar cleanup loop, not just the vectorized main body.

## Checklist

- [ ] Confirmed the workload is elementwise/uniform (same op per element, no per-element
      branching) before expecting SIMD to apply
- [ ] Checked data layout — SoA (or otherwise contiguous, uniformly-typed) rather than AoS —
      for the fields actually used in the hot loop
- [ ] Verified pointer/array aliasing is either provably absent or explicitly annotated
      (e.g., `restrict`) so the compiler is free to vectorize
- [ ] Rewrote data-dependent per-element branches as branchless/masked operations where the
      branch was blocking vectorization
- [ ] For reductions, confirmed the operator is associative and decided whether reassociation
      (e.g., relaxed floating-point flags) is acceptable for this computation
- [ ] Checked a vectorization report or generated assembly before writing manual intrinsics
- [ ] Considered whether the workload's scale actually justifies GPU/cluster parallelism instead
      of (or in addition to) CPU SIMD
