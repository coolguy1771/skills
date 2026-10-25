---
name: javascript-sorting-and-array-mutation-idioms
description: Write correct, idiomatic JavaScript for sorting, shuffling, and swapping array elements — comparator function pitfalls with Array.prototype.sort, in-place vs. copying variants (sort vs toSorted, reverse vs toReversed), stability guarantees, destructuring swaps, and the Fisher-Yates shuffle. Use this when the user's array sorts "wrong" for numbers (e.g. [10, 2, 1] not sorting correctly), asks how to sort objects by multiple fields, wants a "random shuffle" or "randomize this array" in JS, needs a non-mutating sort/reverse, or has a bug caused by two variables referencing the same array after a "copy." Complements binary-search-on-the-answer and the language-agnostic sorting-algorithm-choice knowledge — this skill is about getting JavaScript's own array methods and idioms right, not about which algorithm to pick.
---

# JavaScript Sorting and Array Mutation Idioms

`Array.prototype.sort` is one of the most misused built-ins in JavaScript, almost entirely
because its default behavior surprises people coming from other languages, and because
"sorts the array" and "does not mutate the array" get confused constantly. This skill is about
using JS's own array primitives correctly, not about picking a sorting algorithm (see
`binary-search-on-the-answer` and general algorithm-choice skills for that).

## The single most common bug: default `.sort()` is lexicographic

Without a comparator, `.sort()` converts every element to a string and compares
lexicographically — for numbers, this is almost never what you want:

```js
const a = [22, 9, 60, 12, 4, 56];
a.sort();
console.log(a); // [12, 22, 4, 56, 60, 9] — sorted as strings, NOT numerically

a.sort((x, y) => x - y);
console.log(a); // [4, 9, 12, 22, 56, 60] — correct
```

The comparator contract: return **negative** if `a` should come before `b`, **positive** if `a`
should come after `b`, and **zero** if their relative order doesn't matter. `(a, b) => a - b`
works for numbers because subtraction naturally produces the right sign; for anything else
(strings, dates, objects), write explicit comparisons — don't try to force non-numeric
comparisons into subtraction:

```js
// Wrong: comparing strings by subtracting throws away the actual comparison
names.sort((a, b) => a - b); // NaN for every pair; sort order is undefined/unstable garbage

// Right
names.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));
// or, for locale-aware string comparison:
names.sort((a, b) => a.localeCompare(b));
```

## Multi-field sorting: compare fields in priority order, fall through to the next on ties

```js
const dateNameCompare = (a, b) => {
  if (a.year !== b.year) return a.year - b.year;
  if (a.month !== b.month) return a.month - b.month;
  if (a.day !== b.day) return a.day - b.day;
  return a.name.localeCompare(b.name); // final tiebreaker
};
people.sort(dateNameCompare);
```

Check each field in priority order and return as soon as one differs; only fall through to the
next field on an exact tie. This reads clearly and avoids building a combined sort key string
unless the fields are already string-shaped.

## Stability: guaranteed since ES2019, not before

A stable sort preserves the relative order of elements that compare equal. `.sort()` has been
**required** to be stable since ECMAScript 2019; all evergreen browsers and current Node.js
comply. This matters for exactly the pattern of sorting by one field and then re-sorting by
another (e.g., sort contacts alphabetically, then re-sort so favorites float to the top) —
with a stable sort, the second sort doesn't disturb the alphabetical order within each group:

```js
contacts.sort((a, b) => a.name.localeCompare(b.name)); // alphabetical
contacts.sort((a, b) => (b.isFavorite ? 1 : 0) - (a.isFavorite ? 1 : 0)); // favorites first
// Because sort is stable, alphabetical order survives within "favorites" and "non-favorites"
```

If code must run on a pre-2019 engine (essentially never a real concern today, but worth
knowing), simulate stability by sorting on `[originalKey, originalIndex]` and dropping the index
afterward.

## Mutating vs. non-mutating: know which method you're calling

`.sort()`, `.reverse()`, and `.splice()` all mutate the array **in place** and also return it —
which means chaining works but easily hides the fact that the original array changed:

```js
const original = [3, 1, 2];
const sorted = original.sort(); // mutates `original` AND returns the same reference
console.log(original === sorted); // true — this is the SAME array, not a copy
```

If the original order needs to be preserved (a common requirement when sorting is a derived
view, e.g. for display, while the source array is shared state), use the non-mutating
counterparts introduced in ES2023 — `.toSorted()`, `.toReversed()`, `.toSpliced()`, `.with()` —
or copy first:

```js
const sorted = [...original].sort((a, b) => a - b); // copy, then sort the copy
const sorted2 = original.toSorted((a, b) => a - b);  // ES2023: built-in non-mutating version
```

Check the target runtime's support for `.toSorted()`/`.toReversed()` before relying on them in
code that must run on older Node.js or non-evergreen browsers; `[...arr].sort()` works
everywhere and costs one extra array copy.

## Swapping elements: destructuring, not a temp variable

The idiomatic in-place swap in modern JS uses array destructuring instead of a manual temp
variable — this shows up constantly inside hand-rolled sorting/shuffling/heap code:

```js
[arr[i], arr[j]] = [arr[j], arr[i]];
```

Both sides are evaluated before any assignment happens, so this is safe even when `i === j` or
when the indices reference overlapping structure — no temp variable needed, and it reads as
"these two swap" rather than three separate statements.

## Fisher-Yates shuffle: the correct way to randomize an array

A uniformly random shuffle needs to pick, for each position, a random element from the
**remaining unshuffled** portion — not from the whole array every time (that's a common bug that
produces a biased, non-uniform shuffle):

```js
const shuffle = (arr) => {
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1)); // random index in [0, i], not [0, arr.length)
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr; // mutates arr in place; copy first with [...arr] if that's not wanted
};
```

The loop bound is the critical detail: at step `i`, the random pick must be restricted to `[0,
i]` (the still-unshuffled prefix, inclusive of `i` itself), not `[0, arr.length - 1]` — picking
from the whole array on every iteration is the single most common Fisher-Yates bug and produces
a measurably non-uniform distribution (some permutations become more likely than others), even
though the code looks correct at a glance.

## Binary search: off-by-one and comparator consistency

A hand-rolled binary search in JS has the same classic pitfalls as in any language, plus one JS
specific one — make sure the comparator used to binary-search matches the comparator the array
was actually sorted with (in particular, don't binary search a numerically-sorted array using
default `<`/`>` on non-numeric-looking values, and don't forget `Math.floor` when computing the
midpoint, since JS division always produces a float):

```js
const binarySearch = (sortedArr, target, compare = (a, b) => a - b) => {
  let low = 0, high = sortedArr.length - 1;
  while (low <= high) {
    const mid = low + Math.floor((high - low) / 2); // avoids overflow-style bugs, though less
                                                       // critical in JS (doubles) than in fixed-width int languages
    const cmp = compare(sortedArr[mid], target);
    if (cmp === 0) return mid;
    if (cmp < 0) low = mid + 1; else high = mid - 1;
  }
  return -1; // not found
};
```

## Common pitfalls

- **Calling `.sort()` on numbers with no comparator** and getting lexicographic order.
- **Using `a - b` as a comparator on non-numeric values**, producing `NaN` and undefined sort
  behavior.
- **Assuming `.sort()`/`.reverse()` return a copy** — they mutate in place and return the same
  reference; a caller holding the "original" array sees the sorted result too.
- **Shuffling by picking a random index from the whole array on every iteration** instead of
  shrinking the range — produces a biased (non-uniform) shuffle that looks fine in casual testing.
- **Forgetting a stable sort still requires correct comparator logic** — stability only
  preserves order among elements the comparator calls *equal* (returns 0); it doesn't fix a
  broken comparator.
- **Mixing `.toSorted()`/`.toReversed()` into code that must run on older engines** without
  checking runtime support first.

## Checklist

- [ ] Every `.sort()` call on non-string data passes an explicit comparator
- [ ] Multi-field comparators check fields in priority order and fall through only on exact ties
- [ ] Deliberately chose mutating (`.sort()`) vs. non-mutating (`[...arr].sort()` /
      `.toSorted()`) based on whether the original array must survive unchanged
- [ ] Swaps use destructuring (`[a[i], a[j]] = [a[j], a[i]]`), not error-prone manual temp
      variables
- [ ] Shuffle's random index range shrinks with the loop (`[0, i]`, not `[0, length)`) at every
      step
- [ ] Binary search's comparator matches the comparator the array was actually sorted with
