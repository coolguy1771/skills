---
name: priority-queues-segment-trees-and-union-find
description: Choose and apply the right specialized data structure for repeated queries a hash table and plain arrays can't answer efficiently — a heap (priority queue) for repeated min/max extraction, a segment tree for range queries with point/range updates, or union-find (disjoint-set) for maintaining and merging equivalence classes over time. Use this when the user needs "the current minimum/maximum" repeatedly as items are added/removed, "sum/max/min over a range" with updates in between (range queries + updates), or "are these two things in the same group" with a relationship that keeps merging groups (connectivity, "same community", "same set", union operations). Also use when the user asks "heap vs sorted array", "how do I speed up repeated min-of-array queries", or "how do I efficiently track connected components as edges are added".
---

# Priority Queues, Segment Trees, and Union-Find

No data structure is good for everything — each of these is a narrow specialist that makes one
family of operations fast at the expense of others. Choosing between them (and against a hash
table or plain array) is a matter of naming precisely which operations the problem actually
repeats.

## Choosing among them (and against a hash table)

Ask: **what operation does this problem repeat, and on what?**

| If the repeated question is... | Reach for... |
|---|---|
| "Is X present? What's mapped to X? How many times has X occurred?" | Hash table (see `hash-table-lookup-patterns`) |
| "What's the current minimum/maximum among items I've inserted (some possibly since removed)?" | Heap (priority queue) |
| "What's the sum/min/max over an arbitrary *range* of an array, interleaved with point or range updates?" | Segment tree |
| "Are these two items in the same group? Merge these two groups." (repeated over time as merges happen) | Union-find (disjoint-set) |

A hash table cannot answer "what's the minimum in this collection" without an O(n) scan — that
question is precisely a min-heap's job. A heap cannot answer "what's the max over indices 5
through 17" — that's a segment tree's job, because a heap has no notion of an underlying array
position. Each structure answers a different question fast; matching the *question* to the
*structure* is the whole exercise. There's no universally-fast data structure — accept that and
choose deliberately.

## Heaps (priority queues)

A binary heap keeps the minimum (min-heap) or maximum (max-heap) always at the root, with
insert and extract-min/max both O(log n) — dramatically better than the O(n) it would take to
find a minimum by scanning, or the O(n log n) it would take to keep a fully sorted structure
when you only ever need the extreme value, not the full order.

```python
import heapq

min_heap = []
heapq.heappush(min_heap, 7)
heapq.heappush(min_heap, 3)
heapq.heappush(min_heap, 9)
smallest = heapq.heappop(min_heap)   # 3, in O(log n)
```

Python's `heapq` is min-heap only; for a max-heap, negate values on insert/extract, or store
`(-value, item)` tuples.

**Recognize the heap shape**: any problem where you repeatedly need "the current best (min or
max) of a changing collection" — a scheduling problem picking the next-soonest task, a
simulation always processing the currently-cheapest/most-urgent item next, "k-th
largest/smallest so far" (maintain a bounded heap of size k), or as the priority queue inside
Dijkstra's algorithm (see `graph-search-and-shortest-paths`).

**Complexity**: O(log n) insert and extract-min/max, O(1) peek. Building a heap from n existing
elements is O(n) (not O(n log n)) if done via a proper heapify — worth knowing if you're
initializing from a full array rather than inserting one at a time.

## Segment trees: range queries with updates

A segment tree answers "what's the sum/min/max (or other associative combination) over
indices `[l, r]`" in O(log n), and supports point or range updates in O(log n) too — the
combination that neither a plain array (O(n) per range query) nor a prefix-sum array (O(n) per
update, since every update invalidates everything after it) can deliver together.

The structure: a binary tree over the array where each leaf is one array element and each
internal node stores the combined value (sum/min/max/etc.) of its entire subrange. A query for
range `[l, r]` decomposes into O(log n) precomputed subrange nodes instead of scanning every
element; an update touches only the O(log n) nodes on the path from a leaf to the root.

```python
class SegmentTree:
    def __init__(self, data, combine=min, identity=float('inf')):
        self.n = len(data)
        self.combine = combine
        self.identity = identity
        self.tree = [identity] * (2 * self.n)
        for i, v in enumerate(data):
            self.tree[self.n + i] = v
        for i in range(self.n - 1, 0, -1):
            self.tree[i] = combine(self.tree[2 * i], self.tree[2 * i + 1])

    def update(self, i, value):
        i += self.n
        self.tree[i] = value
        while i > 1:
            i //= 2
            self.tree[i] = self.combine(self.tree[2 * i], self.tree[2 * i + 1])

    def query(self, l, r):               # inclusive range [l, r]
        l, r = l + self.n, r + self.n + 1
        result = self.identity
        while l < r:
            if l & 1:
                result = self.combine(result, self.tree[l]); l += 1
            if r & 1:
                r -= 1; result = self.combine(result, self.tree[r])
            l //= 2; r //= 2
        return result
```

**Recognize the segment-tree shape**: "range sum/min/max query" interleaved with "point (or
range) update," especially when there are many queries and many updates interspersed — if
there were no updates at all, a plain prefix-sum array (O(1) query after O(n) preprocessing) is
simpler and sufficient; segment trees earn their complexity specifically because updates keep
happening between queries.

**Complexity**: O(n) to build, O(log n) per query or update. Both matter — a segment tree is the
wrong choice if you only ever query once after all updates (use prefix sums) or only ever update
without needing range queries (use a plain array/hash table).

## Union-Find (Disjoint-Set)

Union-find maintains a partition of objects into disjoint sets and supports two operations
efficiently over many repeated calls: **Find** (which set is this object currently in?) and
**Union** (merge the sets containing two objects). It is dramatically faster than modeling the
same problem as a graph and re-running a traversal (BFS/DFS) after every new relationship is
learned — see `graph-search-and-shortest-paths` for that heavier alternative, which union-find
exists specifically to avoid needing.

### The three-requirement test for whether union-find applies

Before reaching for union-find, check that the relationship you're maintaining is an
**equivalence relation** — it must satisfy all three:

- **Reflexive**: every object is related to itself.
- **Symmetric**: if x relates to y, then y relates to x (no directionality).
- **Transitive**: if x relates to y, and y relates to z, then x relates to z.

If the relationship fails any of these — most commonly transitivity — a plain Union operation
is unsound: merging x's and y's sets based on "x relates to y" can silently lump together
objects that don't actually satisfy the relationship. ("Enemy of" is a good example of a
relation that's symmetric but neither reflexive nor transitive — see "augmenting" below for how
to still use union-find when the raw relationship isn't an equivalence relation but has useful
structure.)

### Implementation with the two standard optimizations

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.size = [1] * n

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path compression: flatten as we go
            x = self.parent[x]
        return x

    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return False
        if self.size[rx] < self.size[ry]:                  # union by size: attach smaller under larger
            rx, ry = ry, rx
        self.parent[ry] = rx
        self.size[rx] += self.size[ry]
        return True
```

- **Union by size** (or by rank) keeps the resulting trees shallow by always attaching the
  smaller tree under the larger tree's root, rather than the reverse — without it, an adversarial
  sequence of unions can build a degenerate, path-like tree where Find takes O(n).
  - Exception worth knowing: if the problem needs you to remember *which* set's identity/root
    survives a merge in a specific direction (order matters to the answer), you cannot swap
    roots freely based on size — union by size is an optimization that's safe to skip when order
    of merging carries meaning, but otherwise apply it unconditionally.
- **Path compression** makes every node found during a Find point directly (or nearly directly)
  at the root, so future Finds through that node are faster. It's essentially always safe to
  apply — there isn't a known case where it produces wrong answers.
- Together, these two optimizations give amortized near-O(1) (technically O(α(n)), the inverse
  Ackermann function — effectively constant for any n that fits in memory) Find and Union,
  versus O(n) worst case per operation without them.

### Augmenting union-find for relationships that aren't full equivalence relations

A relationship like "enemy of" is symmetric but not reflexive or transitive on its own — yet you
can still use union-find productively by tracking *extra* information alongside each set. The
classic trick ("the enemy of my enemy is my friend"): when told x and y are enemies, you don't
just record that fact — you can also derive that everyone in x's existing friend-set is now an
enemy of everyone in y's existing friend-set (and vice versa), and that everyone in x's
enemy-set is a *friend* of everyone in y's set. Implementing this means each object's union-find
node needs an associated "enemy set" pointer (or parity bit relative to its set representative)
alongside the standard parent/size bookkeeping — the base Find/Union machinery doesn't change,
but what you store and derive per Union call does.

### Recognizing the pattern

- **Directly**: "these two things are now known to be the same group" / "are these two things
  known to be in the same group" — repeated over time, especially when interleaved with
  queries in between merges (if all the grouping information were given up front with no
  queries in between, a single graph traversal to find connected components would suffice and
  union-find's incremental advantage wouldn't matter).
- **Underneath a graph problem**: every union-find problem could be solved (less efficiently) by
  modeling relationships as graph edges and running BFS/DFS to find connected components after
  each addition — union-find is the fast-path replacement specifically for "keep incrementally
  merging groups and answering same-group queries," not a fundamentally different capability.

## Complexity summary

| Structure | Key operation | Complexity |
|---|---|---|
| Heap | insert / extract-min-or-max | O(log n) / O(log n) |
| Heap | build from n elements | O(n) |
| Segment tree | build | O(n) |
| Segment tree | range query / point update | O(log n) / O(log n) |
| Union-Find | Find / Union (with both optimizations) | amortized ~O(α(n)) ≈ O(1) |

## Common pitfalls

- **Reaching for a heap when a range query is actually needed** (or vice versa) — a heap has no
  concept of "index," so "min over indices 5–17" is not a heap operation; that's a segment tree.
- **Building a segment tree when there are no updates** — a prefix-sum array is simpler and
  sufficient; segment trees pay off specifically because of interleaved updates.
- **Applying union-find to a relationship that isn't actually an equivalence relation**, without
  the extra augmentation needed to handle it correctly (the reflexive/symmetric/transitive
  check above) — leads to incorrectly merged groups.
- **Skipping union-by-size or path-compression** — correctness is unaffected, but a long chain
  of adversarial unions degrades Find to O(n), silently turning a near-linear algorithm
  quadratic on unlucky/crafted input.
- **Applying union-by-size when the direction of merging matters to the answer** — swapping
  which root survives a merge can be incorrect for problems that track something order-
  dependent (e.g., which representative was inserted first); check whether this applies before
  optimizing blindly.
- **Modeling a union-find problem as a full graph with repeated traversals** — correct but far
  slower; recognize the equivalence-relation shape early to skip the heavier approach entirely.

## Checklist

- [ ] Named the exact repeated operation (extract-min/max? range query+update? group merge
      + same-group check?) before picking a structure
- [ ] Ruled out a plain hash table/array/prefix-sum as already sufficient before reaching for a
      heap or segment tree
- [ ] For union-find: verified the relationship is reflexive, symmetric, and transitive — or
      identified the extra per-set bookkeeping needed if it isn't
- [ ] Implemented both union-by-size and path compression unless a specific order-dependence
      requirement rules out union-by-size
- [ ] For segment trees: confirmed updates actually happen between queries (otherwise use
      prefix sums instead)
- [ ] Checked complexity end to end (structure cost × number of operations) against the
      problem's constraints
