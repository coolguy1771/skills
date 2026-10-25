---
name: javascript-builtin-collection-performance-tradeoffs
description: Choose correctly among JavaScript's built-in collections — plain Object, Array, Map, Set, and typed arrays (Int32Array, Float64Array, etc.) — based on real performance and correctness tradeoffs, not habit. Use this when the user asks "should I use a Map or an Object", "Set vs array for lookups", "why is my object-as-a-set code buggy or slow", is storing non-string keys, needs guaranteed key iteration order, is doing frequent add/delete on a large collection, or is working with large numeric arrays (matrices, heap arrays, buffers) where a typed array would help. Also load this when reviewing JS code that uses a plain `{}` as a hash map/set and something is behaving unexpectedly (key coercion, prototype properties leaking in, unexpected key order).
---

# JavaScript Built-in Collection Performance Tradeoffs

JavaScript gives you at least four different ways to build a "collection of things," and they
are not interchangeable — each has real correctness gotchas and performance characteristics
that only show up under specific conditions (large size, hot loops, non-string keys). Pick
deliberately.

## Object vs. Map: the decision that trips people up most

Using `{}` as a hash map is extremely common and usually works — until one of these hits:

| Concern | Plain `Object` | `Map` |
|---|---|---|
| Key types | Coerced to string (or Symbol) — `obj[1]` and `obj["1"]` are the same key | Any value, including objects, functions, `NaN` — kept as-is, compared by `SameValueZero` |
| Iteration order | Integer-like keys iterate in **ascending numeric order first**, regardless of insertion order, then string keys in insertion order — a genuinely surprising rule | Guaranteed insertion order, full stop, no special-casing of numeric-looking keys |
| Prototype pollution risk | `"__proto__"`, `"constructor"`, `"toString"` as keys interact with the prototype chain unless you use `Object.create(null)` or check `Object.hasOwn` | No prototype chain involved at all — keys are just entries |
| Size | No O(1) way to get key count; `Object.keys(obj).length` is O(n) | `.size` is O(1) |
| Frequent add/delete | Engines optimize objects around a stable "shape" (see V8 hidden classes below); repeatedly adding/deleting keys defeats that optimization | Designed for exactly this access pattern; no shape-transition penalty |
| JSON / serialization | Native `JSON.stringify` support | Needs manual conversion (`Object.fromEntries(map)`, and only if keys are strings) |

```js
// The classic silent bug: numeric-looking string keys don't preserve insertion order
const obj = {};
obj["b"] = 1;
obj["2"] = 2;
obj["a"] = 3;
obj["1"] = 4;
console.log(Object.keys(obj)); // ['1', '2', 'b', 'a'] — NOT insertion order!

const map = new Map();
map.set("b", 1).set("2", 2).set("a", 3).set("1", 4);
console.log([...map.keys()]); // ['b', '2', 'a', '1'] — actual insertion order, always
```

**Rule of thumb**: reach for `Map` whenever keys aren't fixed, known-in-advance string
identifiers (i.e., whenever you're using the object as a *dictionary* rather than as a
*record/struct* with named fields), whenever keys might not be strings, whenever insertion order
matters, or whenever you'll be adding and removing keys frequently over the object's lifetime.
Reach for a plain object when it's really a fixed-shape record (`{ name, age, email }`) — that's
what objects are actually optimized for.

## Set vs. Array for membership testing

`array.includes(x)` is O(n) — a full scan every call. `set.has(x)` is O(1) average. If a piece
of code calls `.includes(...)` inside a loop (checking membership repeatedly against a growing
or static collection), that's the same "should be a hash lookup" red flag covered generically in
`hash-table-lookup-patterns` — `Set` is JavaScript's built-in way to get that O(1) lookup without
hand-rolling a hash table:

```js
// O(n) per check, O(n^2) overall if this runs inside a loop over many items
const seen = [];
for (const item of items) {
  if (!seen.includes(item)) seen.push(item);
}

// O(1) average per check, O(n) overall
const seen = new Set();
for (const item of items) {
  seen.add(item); // Set already ignores duplicates; no need to check first
}
```

`Set` also dedupes in insertion order and preserves object identity (two distinct objects with
identical contents are *not* the same Set member) — same `SameValueZero` comparison as `Map`
keys, not deep equality. Use a plain array (not a `Set`) when you actually need duplicates,
index-based access, or array methods like `.sort()`/`.map()` chained directly — converting
`Set` to array (`[...set]`) is cheap but is still a conversion step to keep in mind in hot code.

## V8 hidden classes: why object *shape* matters for performance

V8 (and other engines) optimize property access by grouping objects that have the same set of
properties, added in the same order, into an internal "hidden class" (a.k.a. "shape" or "map" —
unrelated to the `Map` built-in). Property lookups on objects sharing a hidden class become
close to as fast as a fixed-offset struct field access. Two things defeat this optimization:

- **Adding properties in different orders across instances**, or adding some properties only
  conditionally:
  ```js
  // Bad: two different hidden classes for what's "the same kind of object"
  const a = {}; a.x = 1; a.y = 2;
  const b = {}; b.y = 2; b.x = 1; // different insertion order -> different hidden class

  // Good: same shape every time
  const makePoint = (x, y) => ({ x, y });
  ```
- **Deleting a property with `delete obj.prop`**, which forces the object into a slower
  "dictionary mode" internally, or setting a property to `undefined` conditionally based on
  runtime branches. If a value is genuinely optional, initialize it to `null`/`undefined` up
  front for every instance rather than adding it later on some code paths and not others.

This matters most for objects created in bulk in a hot loop (thousands of similarly-shaped
records) — for a handful of one-off objects it's not worth worrying about. The practical
takeaway: **construct same-shaped objects through one factory function** (as in `makePoint`
above) rather than building them up field-by-field differently depending on branch, especially
in code processing large collections of records.

## Typed arrays for numeric-heavy structures

A `Float64Array`/`Int32Array`/etc. stores raw fixed-width numbers contiguously in memory, unlike
a regular `Array`, which stores boxed values behind pointers (even when every element is a
number). For large, purely-numeric structures — a binary heap backing array, an adjacency
matrix, a large numeric buffer for signal/image data — a typed array gives:

- Lower memory use (a `Float64Array` of `n` numbers uses `8n` bytes; a regular `Array` of `n`
  numbers uses substantially more, due to per-element boxing overhead).
- Faster iteration in tight numeric loops, because there's no boxing/unboxing and no possibility
  of "holes" or mixed types forcing the engine to de-optimize.
- A **fixed length** — you cannot `push`/`pop`/`splice` a typed array; resizing means allocating
  a new one and copying. This is a real tradeoff, not just a syntax difference: use a typed
  array when the maximum size is known up front (a heap with a capacity bound, an
  `n x n` matrix), and a regular array when the collection genuinely needs to grow/shrink
  dynamically.

```js
// Adjacency matrix for a graph with `n` nodes, weights as 32-bit floats
const n = 1000;
const matrix = new Float32Array(n * n); // flat, contiguous — index [i][j] as matrix[i*n+j]
matrix[i * n + j] = weight;
```

Don't reach for typed arrays by default — they're a specialized tool for numeric bulk data with
a known size ceiling, not a general array replacement (`.map`/`.filter`/`.push` either don't
exist or behave differently on them, e.g. `.map` returns the same typed array type, truncating
non-integer results if the array is an integer type).

## Common pitfalls

- **Using `{}` as a set/map and then being surprised by numeric key reordering** — see the table
  above; this is one of the most common "my code works until keys look like numbers" bugs.
- **Checking `.includes()` in a loop instead of building a `Set` once up front.**
- **Deleting object properties in a hot path**, forcing V8's dictionary-mode fallback, then
  wondering why performance degraded after the object had been "cleaned up."
- **Building same-purpose objects with different property insertion order** across a large
  collection, defeating hidden-class sharing.
- **Reaching for a typed array for a small or dynamically-growing collection**, then fighting
  its fixed length instead of just using a regular array.
- **Assuming `Map`/`Set` use deep equality.** They use `SameValueZero` (like `===`, but treats
  `NaN` as equal to itself and distinguishes `+0`/`-0` in `Set`, not `Map` keys in older specs) —
  two different objects with identical contents are two different keys/members.

## Checklist

- [ ] Keys are non-string, order-sensitive, or the collection has frequent add/delete → `Map`,
      not a plain object
- [ ] Object is really a fixed-shape record with known field names → plain object is fine and
      often faster
- [ ] Repeated membership checks against a growing/static collection → `Set`/`Map`, not
      `array.includes()`
- [ ] Objects of the "same kind" created in bulk are built through one factory function with
      consistent property order (no conditional property addition after construction)
- [ ] Large, purely numeric, fixed-or-bounded-size data → considered a typed array
- [ ] No code relies on `delete` inside a hot loop, or on numeric-string key ordering in a plain
      object
