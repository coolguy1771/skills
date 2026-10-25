---
name: memory-hierarchy-and-cache-optimization
description: Reason about the CPU cache hierarchy (L1/L2/L3, cache lines, locality, associativity, write policies) when explaining why a data structure or access pattern is slower than its algorithmic complexity suggests, or when redesigning data layout for speed. Use this when the user asks "why is iterating this array slower than that one", "why is column-major/row-major traversal slow", "how big should my cache-friendly block/tile be", "what is a cache miss / cache line / false sharing", when reviewing code with strided or pointer-chasing access patterns (linked lists, trees, hash maps with scattered buckets), when comparing array-of-structs vs struct-of-arrays layouts, or when discussing multicore code where independent variables written by different threads seem to interfere with each other's performance (false sharing / cache coherence traffic).
---

# Memory Hierarchy and Cache-Aware Design

RAM is roughly 100x slower than a CPU's registers, so every real CPU sits a small hierarchy of
caches (typically L1, L2, L3 — each roughly an order of magnitude faster than the one below it,
and dramatically smaller) between the core and main memory. Whether a working set fits in a given
cache level, and whether an access pattern *exploits* the cache's design, often matters more for
real-world speed than an algorithm's asymptotic complexity: an O(n) pass with poor locality can
lose badly to an O(n log n) pass that stays cache-friendly, at real-world data sizes. This skill
is about the mechanism precisely enough to predict, from an access pattern, whether it will run
fast or fall off a cache-size cliff.

## The principle underneath every technique here: locality

Caches work because real programs exhibit **locality**:

- **Temporal locality**: a memory location accessed once is likely to be accessed again soon
  (loop variables, hot counters).
- **Spatial locality**: memory locations *near* one just accessed are likely to be accessed soon
  too (array traversal, struct fields).
- **Sequential locality**: a specific access order tends to repeat (walking an array the same
  direction each time).

Caches don't store single addresses — they store fixed-size **cache lines** (a contiguous block
of memory, typically tens of bytes) so that fetching one word also pulls in its spatially-local
neighbors for free. Every technique below is really just "arrange the code/data so it uses what
locality actually gives you, instead of fighting it."

## Reading an access pattern for cache-friendliness

Ask, for the hot loop/data structure in question:

1. **Is the stride small and predictable, or large/pointer-chased?** Walking an array with
   stride 1 (or any small constant stride) means each cache line fetched serves many consecutive
   accesses before being evicted. Walking with a large stride, or via pointers scattered across
   the heap (linked lists, trees, hash tables with poor bucket locality), means each access likely
   touches a *different* cache line — you pay the full miss cost almost every time, and most of
   each fetched cache line is wasted.
2. **Does the working set fit in a given cache level?** If a loop repeatedly touches a working
   set larger than L1 (or L2, or L3), you should expect a visible speed cliff at that size — not
   a smooth degradation. This is the practical reason "the same algorithm gets suddenly slower
   past a certain input size" even with no change in complexity class.
3. **Is data organized the way it's accessed?** Row-major arrays traversed row-by-row (or
   column-major traversed column-by-column) have stride 1; traversing the "wrong" way turns a
   sequential scan into a large-stride scan that defeats the cache line prefetch entirely:

```c
// Row-major C array, size N x N. Cache-friendly: inner loop walks contiguous memory.
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        sum += a[i][j];        // stride-1 access; each cache line serves many iterations

// Same total work, swapped loop order: cache-hostile.
for (int j = 0; j < n; j++)
    for (int i = 0; i < n; i++)
        sum += a[i][j];        // stride-N access; each iteration likely a fresh cache line
```

For large `n`, the second version can be several times slower than the first, despite doing the
exact same arithmetic in the exact same total count — the difference is entirely locality.

## Array-of-Structs vs Struct-of-Arrays

When a hot loop only needs one or two fields of a larger struct, an array-of-structs (AoS) layout
wastes cache line capacity on fields you don't need this pass, while struct-of-arrays (SoA) packs
exactly the needed field contiguously:

```c
// AoS: each cache line pulled in contains position AND velocity AND mass AND ...,
// even if this loop only touches position.
struct Particle { float x, y, z; float vx, vy, vz; float mass; /* ...more fields... */ };
Particle particles[N];
for (int i = 0; i < N; i++) particles[i].x += particles[i].vx;  // vy, vz, mass fetched for nothing

// SoA: the field this loop cares about is fully contiguous — every byte fetched is used.
struct Particles { float x[N], y[N], z[N], vx[N], vy[N], vz[N], mass[N]; };
for (int i = 0; i < N; i++) p.x[i] += p.vx[i];
```

SoA is a real, common technique in performance-sensitive numeric/simulation/game code, and it's
also usually a prerequisite for effective SIMD vectorization (see `simd-and-vectorization-design`)
since SIMD wants contiguous, uniformly-typed lanes. It's not universally better — if code
routinely accesses *all* fields of one record together (as with AoS), AoS keeps that whole record
in one or two cache lines, which SoA would instead scatter across many arrays. Choose the layout
that matches the dominant access pattern, not a fixed rule.

## Blocking/tiling for cache reuse

When an algorithm reuses the same data across nested loops (e.g., matrix multiplication, image
convolution), restructure the loop into fixed-size blocks ("tiles") sized to fit inside a chosen
cache level, so each block's data is loaded once and reused many times before eviction, rather
than being repeatedly evicted and refetched:

```c
// Naive matrix multiply: for large matrices, the inner loop's access to B[k][j]
// strides through memory and thrashes the cache well before N gets large.
for (int i = 0; i < n; i++)
  for (int j = 0; j < n; j++)
    for (int k = 0; k < n; k++)
      c[i][j] += a[i][k] * b[k][j];

// Tiled: process in blocks small enough that each block of a, b, c stays resident
// in cache across the innermost loops, cutting the number of full-row/column refetches.
for (int ii = 0; ii < n; ii += T)
  for (int jj = 0; jj < n; jj += T)
    for (int kk = 0; kk < n; kk += T)
      for (int i = ii; i < min(ii+T, n); i++)
        for (int j = jj; j < min(jj+T, n); j++)
          for (int k = kk; k < min(kk+T, n); k++)
            c[i][j] += a[i][k] * b[k][j];
```

Pick `T` so a tile's working set fits comfortably in the target cache level (commonly tuned for
L1 or L2 based on measurement, not guessed from datasheet size alone — associativity and other
resident data reduce effective usable capacity).

## Cache read/write policy vocabulary worth knowing when reasoning about behavior

- **Direct-mapped**: each address hashes to exactly one cache line slot; cheap but suffers badly
  when two hot addresses alias to the same line (they continually evict each other even if the
  rest of the cache is empty) — a real hazard for data structures whose access pattern happens to
  stride by a power-of-two matching the cache's line-selection hash.
- **Fully associative / set-associative**: an address can live in any line of a set, chosen by
  recency/usage, avoiding the pathological aliasing case above at the cost of more complex
  (parallel) lookup hardware. Set-associative is the practical middle ground used in real CPUs.
- **Write-back vs write-through**: write-back only flushes a modified ("dirty") line to the next
  level down when it's evicted (fast, but a crash/power-loss before eviction loses the update);
  write-through propagates every write immediately (safer, more bus traffic). This mostly matters
  when reasoning about visibility of writes across cores/DMA, not single-threaded performance.
- **Hit rate / miss penalty**: the real cost of a cache design is `miss rate x miss penalty`
  (in cycles), not miss rate alone — an L1 miss serviced by L2 is far cheaper than one that falls
  all the way to DRAM. When profiling, distinguish *which* level is missing, since the fixes
  differ (shrink working set to fit L1/L2 vs. restructure algorithm to reduce total DRAM traffic).

## False sharing (multicore-specific cache pitfall)

Cache coherence between cores operates at cache-line granularity. If two threads on different
cores write to *different* variables that happen to share one cache line, the coherence protocol
treats it as contention on that whole line — the line ping-pongs between cores' caches even
though the threads never touch each other's actual data. This is a common, non-obvious multicore
slowdown:

```c
// Two threads increment separate counters — logically independent, but if counters[0]
// and counters[1] land in the same cache line, every increment invalidates the line
// in the other core's cache: false sharing.
struct { long counters[2]; } shared;   // BAD: counters[0], counters[1] likely share a line

// Pad each hot per-thread counter to its own cache line to eliminate the false dependency.
struct { long counter; char padding[56]; } shared[2];   // pad to typical 64-byte line size
```

Suspect false sharing whenever independent per-thread counters/flags scale worse than expected
as core count increases, especially when each thread's actual working set is otherwise tiny and
non-overlapping.

## Common pitfalls

- **Optimizing algorithmic complexity while ignoring locality at realistic data sizes.** A lower
  big-O algorithm with poor cache behavior can lose to a higher big-O one with good cache
  behavior for the sizes that actually occur in production — measure at real scale, not just
  complexity class.
- **Choosing AoS or SoA by habit rather than by the dominant access pattern.** Profile which
  fields are actually touched together in the hot path before restructuring a data layout.
- **Assuming a larger cache always helps.** If the access pattern itself is non-local (pointer
  chasing through a poorly laid out linked structure), more cache capacity doesn't fix a lack of
  locality — restructure the data structure (e.g., an array-backed tree/heap instead of a
  pointer-linked one) instead.
- **Missing false sharing in multicore code** because each thread's own data looks correct and
  small — the symptom is a scaling problem (worse-than-expected speedup with more cores), not a
  correctness bug, so it's easy to misattribute to something else.
- **Tiling/blocking with a block size guessed from nominal cache size** without accounting for
  associativity, other resident data, or measuring actual performance across candidate sizes.

## Checklist

- [ ] Identified the actual access pattern (stride, pointer-chased, or contiguous) for the hot
      data structure, not just its asymptotic complexity
- [ ] Checked whether the working set fits within L1/L2/L3 at realistic production data sizes
- [ ] Verified loop nesting/traversal order matches the data's memory layout (row-major vs
      column-major, or equivalent)
- [ ] Considered AoS vs SoA based on which fields are actually accessed together in the hot path
- [ ] For loops with heavy data reuse (e.g., matrix/image algorithms), considered tiling to a
      cache-sized block and validated the chosen block size by measurement
- [ ] Ruled out false sharing as the cause of poor multicore scaling before suspecting a logic or
      contention bug elsewhere
- [ ] Distinguished which cache level is actually missing (via profiling) before choosing a fix
