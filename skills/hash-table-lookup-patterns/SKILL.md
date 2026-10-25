---
name: hash-table-lookup-patterns
description: Recognize when a brute-force pairwise-comparison or repeated-linear-scan solution can be replaced with a hash table (hash map/set) for O(1) average lookup, and design that hash table correctly (sizing, hash function, collision strategy). Use this when the user's code has a nested loop comparing every item against every other item (duplicate detection, "have I seen this before," frequency counting, complement search like two-sum, grouping/bucketing by a derived key), when a problem screams "O(n^2) but should be O(n)", or when the user asks how to "speed up a lookup", "deduplicate efficiently", "count occurrences fast", or "avoid an O(n^2) comparison". Also load this when reviewing code for accidental quadratic blowup from linear `in` checks against a list/array inside a loop.
---

# Hash Table Lookup Patterns

The core move: whenever a solution's slowness comes from repeatedly asking "have I seen
something like X before?" or "how many times has X occurred?" by scanning a list, a hash
table turns that scan into an O(1) average-time lookup. This is almost always the first thing to
reach for before anything fancier (sorting, trees, graphs) when the bottleneck is *membership*
or *frequency*, not *order* or *relationship*.

## Recognizing the pattern

Ask two questions about the slow solution:

1. **Is the current approach comparing every item against every other item, or scanning a
   full list/array on each iteration to check membership or find a match?** (nested loops,
   `if x in some_list` inside a loop, repeated linear search for "does this exist already")
2. **Does the problem only care whether something is present / how many times it appeared /
   what value is associated with a key** — not about order, ranges, or relationships between
   elements? If ordering or "find the min/max so far" is what's needed, a hash table is the
   wrong tool (reach for a heap instead — see `priority-queues-segment-trees-and-union-find`).

If both answers are yes, a hash table almost certainly turns an O(n^2) or O(n*m) solution into
an O(n) or O(n+m) one. Canonical shapes:

- **Duplicate / uniqueness detection**: "are there two identical items in this collection?"
  Brute force compares every pair (O(n^2)); a hash set lets you check-and-insert each item
  once (O(n)).
- **Complement search** (two-sum and its relatives): "does some earlier element combine with
  the current one to hit a target?" Instead of a nested loop, hash the elements you've seen so
  far and look up the complement of the current element in O(1).
- **Frequency counting**: "how many times does each value/word/character appear?" A hash
  map from value to count replaces sorting-then-scanning or repeated counting passes.
- **Grouping by derived key**: bucket items by some computed signature (e.g., sorted letters
  of a word to find anagrams) — the hash map's key is the signature, its value is the list of
  items sharing it.
- **"Have I visited this state?"** in search/backtracking — a hash set of visited states avoids
  reprocessing, independent of whether the traversal itself is a graph algorithm.

## The key correctness rule: what does "identical" mean here?

The single most common source of hash table bugs is a hash function that doesn't reflect the
problem's real equality: **if two objects are supposed to compare equal, the hash function
MUST map them to the same bucket** (the converse doesn't need to hold — different objects can
collide into the same bucket, that's just a performance cost, not a correctness bug). Before
writing the hash function, write down explicitly what makes two items "the same" for this
problem:

- Two snowflakes are identical if their (possibly rotated/reflected) sequences of arm lengths
  match → the hash function must be invariant to rotation (e.g., sum the arm lengths mod table
  size) so that rotated duplicates land in the same bucket, then do the expensive exact
  comparison only within that bucket.
- Two words are anagrams if sorted-letters match → hash on the sorted string, not the original.
- A game state repeats if its normalized board representation matches → hash the normalized
  form, not raw coordinates that could represent the same state two different ways.

Get this wrong and the hash table either fails to catch real duplicates (hashed differently even
though "equal") or silently degrades to a wall of collisions.

## Design decisions (language-agnostic)

1. **Table size**: pick something on the order of the expected number of elements (a common
   rule of thumb: 20–100% of the max expected insertions). Too small → long collision chains,
   O(n) buckets, no speedup at all. Too large → wasted memory for little further gain. Size
   doesn't need to be exact; it needs to be in the right order of magnitude.
2. **Hash function**: for simple numeric/short keys, a cheap function (sum mod table size, or
   the language's built-in hash) is usually fine. For strings/composite keys where you want to
   avoid clustering, use a well-mixed function (multiply-and-shift/XOR "avalanche" style, or
   just the standard library's hash — don't hand-roll one for adversarial input unless the
   problem is explicitly adversarial). A fancy hash function does not protect against a crafted
   worst-case input; if that's a real concern (e.g., a public-facing service), that's a different,
   harder problem than typical algorithmic-contest hashing.
3. **Collision strategy**: chaining (each bucket holds a small list) is simpler to implement
   correctly and is the default choice; open addressing (probe forward to the next empty slot)
   saves memory but is fiddlier to get right (especially deletion) and degrades faster as load
   factor rises. In most languages you'll just use the built-in dict/map/set, which already
   makes this decision for you — but understanding it matters when you *must* hand-roll one
   (e.g., a language without a standard hash map, or an interview setting).

## Worked pattern: complement search (two-sum shape)

```python
def has_pair_with_sum(values, target):
    seen = set()
    for v in values:
        complement = target - v
        if complement in seen:      # O(1) average lookup, not a scan of `seen`
            return True
        seen.add(v)
    return False
```

The nested-loop version of this is O(n^2): for each element, scan the rest of the array for its
complement. The hash-set version is O(n): each element is hashed and looked up once. This
same shape generalizes to "does an earlier prefix sum equal current-prefix-sum minus target"
(subarray-sum problems) and "have I seen this normalized state before" (cycle/duplicate
detection in simulations).

## Worked pattern: grouping by derived key

```python
from collections import defaultdict

def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        key = "".join(sorted(w))     # the hash key encodes "what makes these equal"
        groups[key].append(w)
    return list(groups.values())
```

## Complexity tradeoffs

- Average case: O(1) insert/lookup/delete, making an O(n)-or-better overall algorithm possible
  wherever the bottleneck was repeated linear search.
- Worst case: O(n) for a single operation if the hash function collides badly (pathological
  input, or a weak hash function under adversarial data) — this only matters if you don't
  control or trust the input distribution.
- Space: O(n) extra memory for the table — usually an easy trade against turning O(n^2) time
  into O(n) time, but worth noting when memory is the binding constraint.

## Common pitfalls

- **Hashing on the wrong notion of equality** — see above; always state explicitly what makes
  two items the same before writing the hash function.
- **Table too small for the data** — silently turns every operation into an O(k) linked-list walk,
  erasing the entire benefit while looking like a hash table in the code.
- **Reaching for a hash table when the problem needs order** — "what's the minimum so far,"
  "what's in this range," "what's the k-th smallest" are not hash table questions; hashing gives
  you no way to answer them without a full scan. Use a heap, sorted structure, or segment tree.
- **Recomputing a derived key inside the loop unnecessarily** (e.g., re-sorting a string on every
  comparison instead of once when building the group key) — do the expensive part once, then
  let hashing do the repeated work.
- **Forgetting to insert the current element before/after checking**, in complement-search-style
  problems, in a way that lets an element pair with itself when the problem forbids it (check
  whether "seen" should include the current element yet before doing the lookup).

## Checklist

- [ ] Confirmed the bottleneck is membership/frequency lookup, not ordering or range queries
- [ ] Wrote down explicitly what makes two objects "identical" before designing the hash function
- [ ] Hash function guarantees objects considered equal always land in the same bucket
- [ ] Table size is in the right order of magnitude for expected element count
- [ ] Picked chaining unless memory pressure specifically justifies open addressing
- [ ] Verified the complement/grouping key is computed once, not recomputed per comparison
- [ ] Checked self-pairing edge cases in complement-search-style lookups
