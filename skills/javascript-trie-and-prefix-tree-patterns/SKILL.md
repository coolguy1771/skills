---
name: javascript-trie-and-prefix-tree-patterns
description: Implement tries (prefix trees), radix trees, and similar string-keyed digital search trees in JavaScript for autocomplete, prefix search, typeahead, spell-check, or IP-routing-style longest-prefix-match problems. Use this when the user asks to "implement a trie/prefix tree in JS", build "autocomplete" or "typeahead" search, find "all words starting with a prefix", do fast string-keyed lookup where keys share common prefixes, or is deciding between an object/Map of children vs. a fixed-size array of children for a trie node. Distinct from hash-table-lookup-patterns (that's for exact-match lookup with no prefix/ordering structure) — reach for this skill specifically when the "starts with" / prefix relationship between keys is what the problem needs.
---

# JavaScript Trie and Prefix Tree Patterns

A trie stores strings by spreading each key across a path from the root, one character per
edge, rather than storing whole keys in one place the way a hash map does. This makes "find all
keys starting with this prefix" and "does this string exist" both fast and structurally simple —
at the cost of a shape that only makes sense once you've seen it. The key design decision in JS
is **how a node represents its children**, and that choice has real, opposite-direction
performance and memory tradeoffs.

## Two ways to represent a trie node's children in JS

**Fixed array indexed by alphabet position** — fast, dense, wasteful for large/sparse
alphabets:

```js
const EOW = "■"; // end-of-word sentinel; anything not a valid input character works
const ALPHABET = `${EOW}abcdefghijklmnopqrstuvwxyz`;

const newNode = () => ({ links: new Array(ALPHABET.length).fill(null) });
const charIndex = (ch) => ALPHABET.indexOf(ch);
```

Every node allocates a full `links` array (27 slots here; 128+ for full ASCII, unusable for
Unicode) whether or not most of those children exist. Lookup of a specific child is O(1) array
indexing, but memory cost scales with alphabet size × node count regardless of how sparse the
actual data is.

**Object or `Map` keyed by character** — sparse, memory-proportional-to-actual-branching,
slightly slower per-lookup than raw array indexing but rarely the bottleneck in practice:

```js
const newNode = () => ({ links: {} }); // or: new Map()
```

Only characters that actually appear as children get an entry. This is the right default for
large alphabets (Unicode text), for radix trees (see below, where an edge can represent a whole
substring rather than one character, so array-by-single-character-index doesn't even apply), and
for any trie where most nodes have only one or two children (the common case for real text —
most prefixes don't fan out into all 26+ letters).

**Rule of thumb**: fixed array only when the alphabet is small and known (lowercase a–z, DNA
bases, digits) and node fan-out is genuinely dense; object/`Map` otherwise. When in doubt, start
with the object/`Map` version — it's rarely the wrong choice, while a fixed array over a large or
Unicode alphabet is close to always wrong.

## Core trie operations, recursively, one character at a time

Destructuring a string into `[first, ...rest]` gives a clean recursive shape that mirrors linked
list recursion (see `javascript-functional-data-structure-patterns`) — each call peels off one
character and recurses on what's left:

```js
const EOW = "■";

const newTrie = () => null;
const isEmpty = (trie) => !trie;

const find = (trie, wordToFind) => {
  if (!wordToFind) return null;
  return _find(trie, wordToFind + EOW);
};

const _find = (trie, [first, ...rest]) => {
  if (isEmpty(trie)) return null;
  if (first === EOW) return trie.links[EOW] ? trie.links[EOW].data : null;
  return _find(trie.links[first], rest);
};

const add = (trie, wordToAdd, dataToAdd) => _add(trie ?? { links: {} }, wordToAdd + EOW, dataToAdd);

const _add = (node, [first, ...rest], data) => {
  if (first === EOW) {
    node.links[EOW] = { data };
    return node;
  }
  node.links[first] = _add(node.links[first] ?? { links: {} }, rest, data);
  return node;
};
```

The `EOW` (end-of-word) sentinel is what distinguishes "BAD is a complete word" from "BAD is
just a prefix of BADE" — without it, a trie can't tell a stored key apart from a mere prefix of a
longer one. Appending it to every key before recursing, and checking for it explicitly, is the
standard trick; don't skip it even for "just checking prefixes" use cases, because add/find need
a consistent representation of "where words actually end."

## Autocomplete: collecting all completions of a prefix

This is the operation that justifies choosing a trie over a hash map in the first place — a hash
map can't answer "everything starting with 'auto'" without scanning every key:

```js
const collectWords = (trie, prefix = "") => {
  if (isEmpty(trie)) return [];
  let results = [];
  for (const ch of Object.keys(trie.links)) {
    if (ch === EOW) {
      results.push(prefix);
    } else {
      results = results.concat(collectWords(trie.links[ch], prefix + ch));
    }
  }
  return results;
};

const autocomplete = (trie, prefix) => {
  let node = trie;
  for (const ch of prefix) {
    if (isEmpty(node) || !node.links[ch]) return []; // prefix not in trie at all
    node = node.links[ch];
  }
  return collectWords(node, prefix);
};
```

Walk down to the end of the prefix first (O(prefix length)), then collect everything below that
point (O(number of matching words × average length)) — never scan keys that don't share the
prefix, which is exactly the win a flat `Map` of all words can't give you.

## Radix trees: collapsing single-child chains

A plain trie wastes depth on long unbranching chains — if only one word starts with `"e"`,
you'll still walk one node per character of that word. A **radix tree** (compressed trie)
collapses any chain of single-child nodes into one edge labeled with the whole substring instead
of one character:

```js
// Same node shape as an object-based trie — `links` keys are now substrings, not single chars
const newNode = () => ({ links: {} });
```

The add/find logic is structurally the same as the object-based trie, but every comparison
against an edge label now has to check "is this edge a prefix of what I still need to match" or
"is what I still need to match a prefix of this edge" instead of a single-character equality
check, and a mismatched-in-the-middle edge requires splitting it into two shorter edges. This is
worth the added complexity specifically when there are many long shared prefixes and few
branches (this is exactly why IP routing tables and URL routers are typically implemented as
radix trees over path/address segments, not plain tries) — for short keys or high branching
factor, a plain trie is simpler and the depth savings don't matter much.

## Ternary search tries: a hybrid worth knowing about

When the alphabet is large but you still want array-like traversal speed without a trie's memory
overhead, a ternary search trie stores one character per node with three children (`less`,
`equal`, `greater`) instead of one-child-per-letter — effectively a binary search tree over the
current character at each level, chained via the `equal` link for the next character. It trades
a bit more comparisons per level for much better memory use than a fixed-array trie on large
alphabets, without the object/Map overhead of a hash-keyed trie. Reach for this only once a
plain object-keyed trie's memory footprint is a measured problem — it's rarely needed as a first
implementation.

## Common pitfalls

- **Choosing a fixed-size array of children for a large or Unicode alphabet** — memory use
  explodes (every node pays for every possible character slot) for no lookup-speed benefit that
  matters in practice.
- **Forgetting the EOW sentinel**, so a trie can't distinguish a complete word from a prefix of a
  longer one (`BAD` incorrectly reported as present because `BADE` is).
- **Scanning the whole trie for autocomplete** instead of first walking to the end of the prefix
  and only collecting from that subtree onward — defeats the entire point of using a trie.
- **Choosing a radix tree for short, low-branching-factor keys** where the split/merge edge
  logic adds real implementation complexity for no measurable benefit over a plain trie.
- **Using a trie at all when exact-match lookup (no prefix semantics) is the actual
  requirement** — that's `hash-table-lookup-patterns` territory; a `Map` is simpler and faster
  for pure exact-match.

## Checklist

- [ ] Node children representation (fixed array vs. object/Map) matches alphabet size and
      expected fan-out, not chosen by default
- [ ] EOW/end-of-key sentinel is present and checked consistently in add/find/remove
- [ ] Autocomplete/prefix-collection walks to the prefix's end node first, then only traverses
      that subtree — never re-scans from the root per result
- [ ] Considered whether a plain trie's chain depth actually matters before reaching for a radix
      tree's added complexity
- [ ] Confirmed the problem genuinely needs prefix/"starts with" semantics — not just exact-match
      lookup, which a `Map` handles more simply
