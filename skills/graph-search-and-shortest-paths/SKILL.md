---
name: graph-search-and-shortest-paths
description: Model a problem as a graph (nodes + edges/relationships, possibly with cycles) and pick the right shortest-path algorithm — BFS for unweighted/equal-cost edges, 0-1 BFS for two-cost edges, Dijkstra's algorithm for non-negative weighted edges. Use this when the user's problem involves states/positions/objects and moves/relationships between them (grid movement, game states, "fewest moves to reach X", network/route problems, "shortest path", "minimum steps"), when a DP-style recursive formulation turns out to have cycles in its subproblem dependencies, or when the user asks "BFS vs Dijkstra", "how do I find the shortest path", or "why is my recursion looping forever". Complements dynamic-programming-problem-recognition (use that instead when the subproblem dependency graph is acyclic) and priority-queues-segment-trees-and-union-find (Dijkstra needs a priority queue).
---

# Graph Search and Shortest Paths

Model anything as a graph the moment a problem is about a **set of objects and relationships
between them** — board positions, cities, people, courses, words — and you unlock a large
family of well-understood, fast algorithms instead of ad hoc reasoning per problem. The two
workhorses here are breadth-first search (BFS) for unweighted shortest paths and Dijkstra's
algorithm for non-negative weighted shortest paths.

## Recognizing that a problem is a graph problem

- The problem is fundamentally about **a set of objects and pairwise relationships/moves**
  between them: board squares reachable by valid moves, cities connected by roads, courses
  connected by prerequisites, people connected by friendship.
- You want the **minimum number of steps/moves/edges**, or the **minimum total cost**, to get
  from one state to another (or to all other states).
- **The cycle test against DP**: if you tried to write this as a recursive/DP decomposition and
  found that subproblems can depend on each other in a cycle — reaching state A can lead back
  to a previously-seen state A again through a different sequence of moves — that's the signal
  it's a graph problem, not DP. (See `dynamic-programming-problem-recognition` for the reverse
  direction of this same test.) None of the DP problems in a typical DP problem set have this
  property — their subproblems always strictly shrink (fewer minutes, fewer items). The moment
  "fewer" stops being guaranteed, you likely need a graph.

## Graphs vs. trees

Every tree is a graph, but graphs are more general in three ways, each of which matters for
which algorithm applies:

- **Cycles**: graphs can have them, trees can't. A cycle is exactly the signal from the test
  above that plain recursion/DP won't terminate correctly.
- **Direction**: edges can be one-way (a prerequisite relationship: you can go from the
  prerequisite to the course, not the reverse) or two-way. BFS and Dijkstra both work
  unmodified on directed or undirected graphs — direction just constrains which edges you're
  allowed to traverse.
- **Disconnection**: a graph need not have a path between every pair of nodes. Algorithms that
  assume connectivity (and only run search from one starting node) will silently miss
  unreachable parts — decide up front whether "unreachable" is a valid/expected outcome for
  your problem and handle it explicitly (e.g., return -1 or infinity for unreached nodes).

## BFS: unweighted (or equal-cost) shortest paths

BFS explores nodes in increasing order of distance (number of edges) from the start, which is
exactly what makes it a correct shortest-path algorithm when every edge has the same cost:

```python
from collections import deque

def bfs_shortest_distances(start, adjacency):
    dist = {start: 0}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in adjacency[node]:
            if neighbor not in dist:               # first time we reach it = shortest distance
                dist[neighbor] = dist[node] + 1
                queue.append(neighbor)
    return dist
```

**Why it's correct**: BFS processes nodes in non-decreasing distance order, so the first time
any node is reached, it's via a shortest path — there's no way a later-discovered path could be
shorter. This is also why a plain `visited` check (rather than re-relaxing) is enough: once
found, a node's BFS distance never improves.

**Runtime**: O(V + E) — each edge is examined once to see if it leads to an undiscovered node.
The practical lever for keeping BFS fast isn't unweighted-vs-weighted, it's **how many times
you invoke BFS** and **how many edges the graph has**; an algorithm that calls BFS from every
node (O(V) invocations) turns an O(E)-per-call algorithm into O(V·E) overall — look for ways to
answer the real question with a single BFS (or a small constant number of them) before
accepting a per-node BFS as the final solution.

### 0-1 BFS: when edges cost 0 or 1 (not just uniform 1)

If every edge costs either 0 or 1 (not a general weight), you don't need Dijkstra's full
machinery — a double-ended queue keeps BFS's O(V+E) speed:

```python
from collections import deque

def zero_one_bfs(start, adjacency):   # adjacency[node] = list of (neighbor, cost) with cost in {0,1}
    dist = {start: 0}
    dq = deque([start])
    while dq:
        node = dq.popleft()
        for neighbor, cost in adjacency[node]:
            new_dist = dist[node] + cost
            if neighbor not in dist or new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                if cost == 0:
                    dq.appendleft(neighbor)   # 0-cost edge: keep at the front, same "layer"
                else:
                    dq.append(neighbor)       # 1-cost edge: goes to the back, next "layer"
    return dist
```

This is the right middle ground before jumping straight to Dijkstra — recognize the "only two
distinct edge costs" shape (e.g., "free to turn, costs 1 to move forward") as its own case.

## Dijkstra's algorithm: non-negative weighted shortest paths

When edges have arbitrary non-negative weights, use a priority queue (min-heap) to always
expand the currently-closest undetermined node next — see
`priority-queues-segment-trees-and-union-find` for the heap itself:

```python
import heapq

def dijkstra(start, adjacency):   # adjacency[node] = list of (neighbor, weight)
    dist = {start: 0}
    pq = [(0, start)]
    while pq:
        d, node = heapq.heappop(pq)
        if d > dist.get(node, float('inf')):
            continue                       # stale entry; a better distance was already found
        for neighbor, weight in adjacency[node]:
            new_dist = d + weight
            if new_dist < dist.get(neighbor, float('inf')):
                dist[neighbor] = new_dist
                heapq.heappush(pq, (new_dist, neighbor))
    return dist
```

**Runtime**: O((V + E) log V) with a binary heap — each edge can trigger one heap push, and
each push/pop is O(log V).

**Why it requires non-negative weights**: Dijkstra finalizes a node's distance the moment it's
popped from the priority queue, on the assumption that no future (necessarily longer, since the
queue is sorted by distance) path could ever beat it. A negative edge breaks that assumption — a
longer-looking path could later take a negative edge and become shorter than an
already-finalized one. If negative weights are possible, Dijkstra silently gives wrong answers
rather than failing loudly — check for this before trusting a weighted shortest-path result. (A
different algorithm, such as Bellman-Ford, is needed for graphs with negative edges; that's out
of scope here, but the warning sign is worth knowing.)

## Building the graph from the problem statement

Often the hardest part isn't the search — it's recognizing what the nodes and edges *are* when
they aren't handed to you directly:

- **Board/grid problems**: nodes are cells (or cell + extra state, like "cell and direction facing"
  if direction matters to what moves are legal next); edges are legal moves between them.
- **Adjacency list vs. adjacency matrix**: a list (map from node to its neighbors) is generally
  preferred — O(V + E) space and lets you iterate a node's neighbors in time proportional to its
  degree. A matrix (V×V grid of weights/booleans) is worth it only when the graph is dense or
  when O(1) "is there an edge between these two specific nodes" lookups matter more than
  memory — it costs O(V^2) space regardless of how many edges actually exist.
- **State-space graphs**: when "nodes" aren't literally given (e.g., "number of pieces of a
  divided item," "position plus remaining resource"), the graph is implicit — you generate a
  state's neighbors on demand from the rules of the problem rather than reading an edge list.

## Counting shortest paths (not just their length)

A frequent extension: not just "what's the shortest distance" but "how many distinct shortest
paths are there." The trick is to track a count alongside each distance and update it during
relaxation: when you find a strictly shorter path to a node, reset its count to the new path's
count; when you find an *equally* short alternative path, add to the existing count rather than
replacing it. This works with both BFS (equal-weight edges) and Dijkstra (weighted edges) —
the relaxation step just gets a second piece of bookkeeping alongside the distance update.

## Common pitfalls

- **Re-running BFS/Dijkstra from every node when the question only needs answers relative to
  one or a few sources** — turns O(V+E) into O(V(V+E)); look for the reformulation that needs
  only a constant number of search invocations (e.g., search from the *target* instead of every
  source, or add a virtual source node connected to all real starting points).
- **Using Dijkstra on a graph with negative edges** without checking — gives silently wrong
  answers, not an error.
- **Using full Dijkstra when every edge has the same weight (or only two distinct weights)** —
  correct but needlessly slower than BFS / 0-1 BFS; recognize the special case.
- **Adjacency matrix on a sparse graph** — wastes O(V^2) memory and time scanning
  nonexistent edges when an adjacency list would be O(V+E).
- **Treating a graph problem as a tree/DP problem** — leads to infinite recursion or wrong
  answers on any cyclic input; run the cycle test before committing to a recursive DP
  formulation.
- **Forgetting to handle unreachable nodes** — a `dist` map/array that's silently missing an
  entry (or defaults to 0) for an unreached node gets misread as "distance zero" instead of
  "unreachable."

## Checklist

- [ ] Confirmed the problem is about objects + relationships/moves, and identified what the
      nodes and edges actually are (possibly implicit/generated on demand)
- [ ] Ran the cycle test: if subproblems can revisit a prior state, it's a graph problem, not DP
- [ ] Chose adjacency list unless the graph is dense or O(1) edge-existence lookups are needed
- [ ] Picked BFS for uniform edge cost, 0-1 BFS for exactly two distinct edge costs, Dijkstra for
      general non-negative weights
- [ ] Verified no negative edge weights before using Dijkstra
- [ ] Checked how many times the search algorithm is invoked overall — a per-node loop around
      a linear-time search silently changes the total complexity by a factor of V
- [ ] Handled unreachable nodes explicitly rather than letting them default to a misleading value
