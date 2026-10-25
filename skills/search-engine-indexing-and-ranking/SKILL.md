---
name: search-engine-indexing-and-ranking
description: Explain or design how search systems find and rank records quickly at scale — indexing (fixed vs. variable-size storage, binary search, hash tables with collision handling), web crawling, and link-based ranking (PageRank-style authority scoring). Use this when the user asks "how do search engines work", "how does an index make lookups fast", "why can't I just do a sequential/linear scan over all my data", "how does PageRank work", "how do I design a search feature / autocomplete / full-text search for my app", is choosing between a hash table and a sorted/indexed structure for lookups, is implementing or debugging hash-table collision handling or deletion (tombstones), or is reasoning about how to rank results by relevance/authority rather than raw keyword count. Also trigger for "what is an inverted index", crawler/sitemap questions, or "why is naive keyword-count ranking easy to game".
---

# Search Engine Indexing and Ranking

Search has two separable problems that are easy to conflate: **finding** candidate records
fast (indexing) and **ranking** them by relevance once found (scoring). A system can be fast
and useless (finds everything containing the word "cake" instantly, in no useful order) or
well-ranked and useless (perfect relevance scoring over a sequential scan that takes ten
seconds). Real search systems solve both, and the two problems have almost entirely different
tools: indexing is a data-structure problem, ranking is a signal-design problem.

## The storage-allocation tradeoff indexing exists to solve

Fast lookup by position (e.g., "jump straight to record 500,000") requires **fixed-size
records** stored contiguously, because the address of record N is simply
`base_address + N * record_size` — no scanning required. But most real data (names, documents,
song files) varies in size, and fixed-size allocation for variable-size data means either
wasting space (padding everything to the maximum size) or truncating data that doesn't fit.

**Indexing** resolves this by splitting the record: store a compact, fixed-size `(key,
address)` pair in the index, and store the actual variable-size payload wherever convenient,
referenced by that address. The index — small, fixed-size, and therefore searchable with
binary search or hashing — can live in fast memory even when the bulk data it points to lives
on slower storage. This is the mechanism behind essentially every database index, filesystem
directory, and search engine: **you never binary-search or hash the big variable-size blobs
directly; you search a small fixed-size index that points at them.**

## Binary search: why sorted + fixed-size beats sequential scan

A sequential scan examines on average half the records to find a match (and all of them to
confirm absence) — O(n). **Binary search** on sorted, randomly-addressable data halves the
remaining search space every comparison — O(log n). The difference is not cosmetic at scale:
a sequential search of 1,000,000 records examines ~500,000 on average; binary search examines
at most 20. Binary search requires two preconditions that are easy to forget when debugging a
"why is my lookup slow" issue: the data must be **sorted**, and individual records must be
**randomly addressable** (fixed-size or index-mediated) — binary search over a linked list or
other sequential-access structure degrades back to linear time because "jump to the midpoint"
isn't actually O(1) there.

```python
def binary_search(sorted_keys: list, target) -> int | None:
    lo, hi = 0, len(sorted_keys) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if sorted_keys[mid] == target:
            return mid
        elif sorted_keys[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return None
```

The cost binary search hides: keeping data sorted as records are added/removed. For
data with frequent insertions/deletions, the cost of re-sorting after every change can exceed
the lookup-speed benefit — which is exactly when hash tables become the better structure.

## Hash tables: O(1) lookup, at the cost of order

A hash function maps a key directly to a storage slot, so lookup doesn't require sorted order
or scanning at all — average O(1). This makes hash tables the right structure for **dynamic**
collections (frequent add/remove), where re-sorting overhead would dominate. The tradeoffs
that come with this speed:

- **Collisions are inevitable** — the pigeonhole principle guarantees two different keys will
  sometimes hash to the same slot once the table has more possible keys than slots. Two
  standard resolutions: **open addressing** (probe forward to the next empty slot; this is
  what most in-language hash tables use) or **chaining** (each slot holds a list of all
  entries that hashed there). Open addressing means a lookup for a collided key might have to
  scan several slots past its "home" slot before finding it or an empty slot — degrading
  toward a sequential search as the table fills up, which is why load factor matters.
- **Deletion under open addressing needs tombstones, not just clearing the slot.** If key A
  collided into key B's home slot and you delete B by marking its slot merely "empty," a
  later search for A stops at that now-empty slot and incorrectly reports A missing — even
  though A is still stored later in the probe sequence. The fix: mark deleted slots as
  **tombstones** (occupied-but-empty) so searches keep probing past them, while insertions may
  reuse a tombstone slot for a new entry.
- **Rehashing** — once a table gets too full (collisions dominate, degrading toward linear
  search, and tombstones accumulate without ever being reclaimed), rebuild into a larger table
  and re-insert every live key. This is an amortized cost, not a per-operation one; if you see
  a hash table implementation with no rehashing/resize logic, that's a latent
  performance-cliff bug for any workload with sustained growth.

```python
class HashTableWithTombstones:
    EMPTY, TOMBSTONE = object(), object()

    def __init__(self, size=16):
        self.slots = [self.EMPTY] * size

    def _probe(self, key):
        i = hash(key) % len(self.slots)
        first_tombstone = None
        while self.slots[i] not in (self.EMPTY,):
            if self.slots[i] is self.TOMBSTONE and first_tombstone is None:
                first_tombstone = i
            elif self.slots[i] is not self.TOMBSTONE and self.slots[i][0] == key:
                return i, True
            i = (i + 1) % len(self.slots)
        return (first_tombstone if first_tombstone is not None else i), False

    def set(self, key, value):
        i, found = self._probe(key)
        self.slots[i] = (key, value)

    def get(self, key):
        i, found = self._probe(key)
        return self.slots[i][1] if found else None

    def delete(self, key):
        i, found = self._probe(key)
        if found:
            self.slots[i] = self.TOMBSTONE   # NOT self.EMPTY — see explanation above
```

**Hash table vs. sorted index — the actual decision**: choose hashing for point lookups
("does this exact key exist, and what's its value") on frequently mutated data. Choose a
sorted/indexed structure when you need **range queries** ("everything between A and M") or
ordered iteration — hash tables give you neither, since the hash function deliberately
scatters related keys apart.

## Inverted indexes: the data structure behind full-text search

A search engine's core structure is an **inverted index**: instead of `document -> words it
contains`, store `word -> list of documents containing it` (plus enough position/frequency
data to support phrase queries and ranking). Building one is conceptually: crawl/collect
documents, tokenize each into words, and for each word append `(document, position)` to that
word's posting list. Looking up a query becomes intersecting the posting lists of its
constituent words — fast because each posting list is itself an indexed, sorted structure.
Real engines also index **stemmed/normalized forms** and known synonyms (`insomnia` ~
`sleeplessness`) so a query matches documents that use different wording for the same concept
— naive exact-token matching alone misses a large fraction of relevant documents.

## Ranking: why raw keyword count is not enough, and what replaces it

A naive ranking ("show the page where the query term appears most") is trivially gamed:
stuffing a page with invisible repeated keywords, or copying legitimate content onto a
spam/ad-revenue site, both score well under pure term frequency despite being useless or
fraudulent results. The fix used by real search engines is to stop trusting a page's
**self-reported** content signal alone and add an **externally-verified authority signal**:
other pages' links to it, which the page being ranked doesn't control.

The key refinement (the idea behind PageRank) is that not all inbound links count equally — a
link from a page that itself has many inbound links should carry more weight than a link from
an obscure or newly created page. This is naturally modeled as an iterative propagation
problem: each page's authority score is a function of the authority scores of the pages
linking to it.

```python
def pagerank(links: dict[str, list[str]], damping=0.85, iterations=50) -> dict[str, float]:
    """links: page -> list of pages it links to. Returns a relative authority score per page."""
    pages = list(links)
    n = len(pages)
    scores = {p: 1 / n for p in pages}
    inbound = {p: [src for src, outs in links.items() if p in outs] for p in pages}
    for _ in range(iterations):
        new_scores = {}
        for p in pages:
            contribution = sum(
                scores[src] / max(len(links[src]), 1) for src in inbound[p]
            )
            new_scores[p] = (1 - damping) / n + damping * contribution
        scores = new_scores
    return scores
```

This is exactly what defeats naive keyword stuffing and simple link farms: a spam page can
control its own content and can create many low-authority pages linking to itself, but it
cannot manufacture links *from* pages that already have high authority (which is precisely
what independently-earned inbound links from the rest of the web represent) — the damping
factor and iterative propagation mean authority only accumulates through genuinely
well-connected structure, not through self-created volume.

## Common pitfalls

- **Binary-searching or hashing an unsorted or non-randomly-addressable structure** (e.g., a
  linked list) and being surprised the expected speedup doesn't materialize — both techniques
  have structural preconditions, not just algorithmic ones.
- **Clearing a hash table slot on deletion instead of tombstoning it** — silently breaks
  lookups for any other key that collided through that slot, and the bug is often intermittent
  (only shows up when there was a prior collision on that exact slot).
- **Choosing a hash table for a workload that actually needs range queries or ordered
  iteration** — hash tables deliberately scatter keys; if you find yourself iterating a hash
  table and sorting the results afterward every time, a sorted index was probably the right
  primary structure.
- **Ranking purely by term frequency/keyword match** in any search feature — trivially gamed
  and doesn't reflect actual relevance or trustworthiness; add at least one externally-
  verified signal (links, citations, user click-through, purchase/engagement history)
  alongside content matching.
- **Ignoring word-form variation (stemming/synonyms) in a full-text search feature** — users
  searching "running shoes" expect to match documents saying "run" or "runner," and a
  literal-token-only index will silently under-return relevant results without ever erroring.
- **Never rehashing/resizing a hash table under sustained growth** — collisions and probe
  chains grow with load factor, degrading toward sequential-search performance exactly when
  the dataset is largest and speed matters most.

## Checklist

- [ ] Confirmed whether the workload needs point lookups (favor hashing) or range/ordered
      queries (favor a sorted/indexed structure) before picking a data structure
- [ ] Verified hash table deletions use tombstones, not slot-clearing, when using open
      addressing
- [ ] Confirmed the index separates small fixed-size key/address pairs from variable-size
      payload data, rather than trying to directly index variable-size records
- [ ] Checked that ranking uses at least one externally-verified signal beyond raw
      keyword/term frequency
- [ ] Considered word-form normalization (stemming, synonyms) for any full-text search feature
- [ ] Has a rehashing/resize strategy for hash tables under sustained growth, not just a fixed
      initial size
