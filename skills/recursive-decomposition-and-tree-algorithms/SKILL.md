---
name: recursive-decomposition-and-tree-algorithms
description: Design a correct recursive solution over a tree (or naturally self-similar/nested structure) by decomposing a problem into independent subproblems on its children, and know when plain recursion is enough versus when it needs memoization. Use this when the user is processing a tree/nested/hierarchical structure (file trees, org charts, parse trees, nested JSON, binary trees), asks "how do I write this recursively", is debugging a recursive function that's wrong at the base case or double-counts/misses nodes, or is deciding between recursion and an iterative approach. Distinct from dynamic-programming-problem-recognition — reach for this skill first for tree/hierarchy problems where subproblems (children) don't overlap; escalate to the DP skill only once you notice the same subproblem being recomputed from multiple callers.
---

# Recursive Decomposition and Tree Algorithms

Recursion is the natural fit whenever a problem's structure is *self-similar*: solving it requires
solving the same kind of (smaller) problem on its parts, and combining those results. Trees are
the cleanest example — a tree is defined recursively (a node plus subtrees, which are
themselves trees) — but the technique generalizes to any nested structure: filesystem trees,
nested comments/replies, parse trees, recursive data formats.

## Recognizing the pattern

- The input is naturally hierarchical/nested (a tree, a nested list, a recursive grammar), OR
  the problem can be *reframed* as one (e.g., "shortest path with limited allowed moves" often
  isn't a tree, but plenty of divide-and-conquer problems are).
- The problem can be answered for a whole structure by answering it for each child/subpart
  and combining the answers (sum, max, concatenation, boolean OR/AND, etc.).
- There's an obvious **base case** — a leaf node, an empty structure, a size-0/1 input — where
  the answer is trivial and doesn't need further decomposition.

If you can state the problem as "the answer for this node depends only on the answers for its
children," you have a recursive decomposition. Write that sentence down explicitly before
writing any code — it *is* the recursive case.

## The three-part recipe

Every correct recursive function over a tree/nested structure has three parts, and bugs almost
always trace back to one of them being wrong or missing:

1. **Base case(s)**: the smallest input(s) the function must handle without recursing — an
   empty tree, a null child, a leaf. Missing a base case is the #1 cause of infinite recursion or a
   crash on null; handling it *incorrectly* (e.g., returning 0 instead of a sentinel like `-infinity`
   for a "max" aggregation) silently corrupts every answer built on top of it.
2. **Recursive case**: call the same function on each child/subpart to get their answers.
   Independence matters here — the children's subproblems must not depend on each other's
   results, or you don't have simple recursion (see "When recursion is enough" below).
3. **Combine step**: turn the children's answers plus the current node's own data into this
   node's answer (sum them, take a max, concatenate, etc.). This is the part most often
   forgotten when a problem needs more than "just recurse and return" — e.g., you need to add
   the current node's own value on top of what the children returned.

```python
class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

def total_value(node):
    if node is None:                          # base case
        return 0
    total = node.value                        # this node's own contribution
    for child in node.children:
        total += total_value(child)           # recursive case (independent subproblems)
    return total                              # combine step
```

## Reading recursive input correctly

A common practical snag with tree problems isn't the algorithm — it's building the tree from
whatever flat representation the input gives you (parent pointers, an edge list, indented text).
Get this step right and isolated from the recursive logic:

1. Parse the flat representation into an explicit structure (an array of child-lists indexed by
   node ID is usually simplest — `children[i]` is the list of children of node `i`).
2. Identify the root(s) explicitly (often the node with no parent, or given directly in the input).
3. Only then run the recursive algorithm over that structure. Debugging is much easier when a
   malformed tree and a wrong recursive formula are separable failures.

## Two independent recursions in one problem

A subtlety worth naming: a single problem can require *two separate* recursive computations
that both traverse the same tree but compute different things — e.g., one recursion computes
"total candy collectable if I only ever move downward from this node," and a second, different
recursion computes "the best answer overall, considering paths that go up through a node and
back down another branch." Don't force one recursive function to do both jobs; write each as
its own clean base-case/recursive-case/combine, then compose them (the second often calls the
first as a helper at each node).

## When recursion alone is enough vs. when it needs memoization

Recursion by itself is correct and fast **as long as each subproblem is solved only once**. For
tree decomposition specifically, this is almost always true: the subtree rooted at a node's left
child and the subtree rooted at its right child are completely independent — solving one never
requires re-solving the other, and no third caller will ever ask for either of those exact
subproblems again. That's why "total candy in a tree" or "number of descendants of each node"
need no memoization at all — plain recursion is already linear in the number of nodes.

**The moment you notice the same subproblem gets asked for by multiple different call paths**
(a classic sign: the recursion isn't decomposing a tree but exploring choices with numeric
parameters, like "ways to make total T," where two different sequences of choices land on the
same remaining T), you've left this territory and entered dynamic programming — see
`dynamic-programming-problem-recognition` for that next step. A quick test: if you could draw
the recursive calls as a tree and no node ever repeats, plain recursion is enough; if the call
graph has the same subproblem reachable via multiple paths, it's DP.

## Complexity

Plain tree recursion where each node is visited once and does O(1) (or O(k) for its own
children count) work is O(n) overall for n nodes — the recursion tree has exactly n nodes,
mirroring the input. Watch for solutions that *look* linear but recompute a subtree's answer
from multiple ancestors (e.g., naive "height of every node" implementations that recompute
height from scratch at each node) — that turns O(n) into O(n^2) even though no memoization
is obviously "missing"; the fix is usually to have one traversal compute and return the needed
value for reuse, rather than a fresh top-down recomputation per query.

## Common pitfalls

- **Wrong or missing base case value** — returning `0` for a base case in a `max`-aggregation
  problem instead of a proper "no valid answer" sentinel, silently making an empty branch look
  better than it is.
- **Forgetting the combine step's own-node contribution** — recursing into children correctly
  but forgetting to add/incorporate the current node's own data before returning.
- **Off-by-one in "distance walked" or "depth" style accumulators** — decide once whether the
  root counts as depth 0 or depth 1, and thread that consistently through every recursive call.
- **Silently quadratic recursion** — recomputing a subtree property from scratch at every
  ancestor instead of having the recursion return and reuse it once.
- **Treating a graph like a tree** — plain recursion assumes no cycles and no node reachable two
  different ways; if the structure isn't actually a tree (a general graph, a DAG with shared
  nodes), see `graph-search-and-shortest-paths` instead, or detect and memoize once subproblems
  can overlap.
- **Stack depth** on very deep/unbalanced trees (a long chain rather than a balanced tree) —
  recursion depth equals tree height, which can hit language recursion limits; convert to an
  explicit stack-based iterative traversal if depth could be large (thousands+).

## Checklist

- [ ] Can state in one sentence: "the answer for a node depends only on the answers for its
      children" (or identify why this problem needs a second, different recursion too)
- [ ] Base case handles the smallest/empty input with the *correct* sentinel value, not just "no
      crash"
- [ ] Combine step incorporates the current node's own contribution, not just children's results
- [ ] Confirmed subproblems (children) are independent — no subtree's answer is needed by
      another subtree's computation
- [ ] Verified no subproblem is solved more than once anywhere in the call tree; if it is, moved
      to `dynamic-programming-problem-recognition`
- [ ] Tree-building/parsing step is isolated and tested separately from the recursive logic
- [ ] Considered recursion depth vs. tree height for very deep/skewed inputs
