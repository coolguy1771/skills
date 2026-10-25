---
name: javascript-persistent-and-immutable-data-structures
description: Build immutable/persistent data structures in JavaScript (lists, stacks, queues, trees) using structural sharing so every "update" returns a new version without copying the whole structure. Use this when the user asks for an "immutable data structure in JS", wants to avoid mutating shared state (Redux/React state updates, undo/redo history, time-travel debugging, concurrent access without locks), asks "how do I update this without mutating the original array/object", is implementing an undo stack or version history, or has a bug caused by two references pointing at the same mutable object/array. Distinct from javascript-functional-data-structure-patterns (that's about closures vs. classes for encapsulation, independent of mutability) — reach for this skill specifically when immutability itself, not just functional style, is the requirement.
---

# JavaScript Persistent and Immutable Data Structures

An immutable data structure never changes after creation; "modifying" it produces a new
structure instead. Done naively (`[...array, newItem]`, `{...obj, key: value}`) this is easy for
small objects but means copying the entire structure on every update — O(n) per change, which
is wasteful for large or frequently updated structures. **Structural sharing** (a.k.a. a
*persistent* data structure) fixes this: a new version reuses every part of the old structure
that didn't change, and only rebuilds the path from the root to the change. This is the same
idea React/Immer/Immutable.js rely on internally, and it's straightforward to implement by hand
for lists, stacks, and trees.

## The core trick: only rebuild the path that changed

For a singly linked list `{ value, next }`, adding or removing at the **front** is already O(1)
and fully persistent — the new list just points its `next` at the old list, which is completely
unaffected and safe to keep using from elsewhere:

```js
const push = (list, value) => ({ value, next: list }); // O(1), old `list` untouched

const oldList = push(push(push(null, "c"), "b"), "a"); // a -> b -> c
const newList = push(oldList, "z");                     // z -> a -> b -> c
// oldList is still a -> b -> c, completely valid and unaffected by newList existing
```

Modifying or removing an element **deeper** in the list requires rebuilding every node from the
root down to (and including) that point, but nodes *after* the change point are shared,
untouched, between old and new versions:

```js
// Remove the element at `position`, functionally: only nodes before it are recreated
const remove = (list, position) => {
  if (list === null) return list;
  if (position === 0) return list.next;                       // share the rest, unchanged
  return { value: list.value, next: remove(list.next, position - 1) }; // rebuild just this node
};
```

This is O(n) in the worst case (removing near the tail rebuilds almost everything), but for
edits near the front it degrades gracefully to close to O(1) — and crucially, **any other
reference to the original list remains valid and correct**, forever. That's the entire value
proposition: cheap, safe snapshots.

## Stacks: already persistent for free

A stack built the way described in `javascript-functional-data-structure-patterns` (a linked
list restricted to one end) is *automatically* a persistent structure — push and pop both only
touch the top node:

```js
const push = (stack, value) => ({ value, next: stack });
const pop = (stack) => stack.next; // returns a NEW stack reference; `stack` itself is untouched
```

Every prior version of the stack remains a valid, independent snapshot. This is exactly how
undo history or time-travel debugging can be implemented cheaply: keep an array of `stack`
references, one per action, without deep-copying anything on each step.

## Queues: the hard case, solved with two stacks

A naive persistent queue (append at the tail of a linked list) requires rebuilding the entire
list on every enqueue, because in a singly linked list you can't reach the tail without
rebuilding every node's `next` pointer along the way. The standard trick is representing a queue
as **two stacks** — a `backPart` (freshly enqueued items, most recent on top) and a `frontPart`
(items ready to dequeue, oldest on top):

```js
const newQueue = () => ({ backPart: null, frontPart: null });

const enter = (queue, value) => ({
  backPart: { value, next: queue.backPart }, // O(1) persistent push onto backPart
  frontPart: queue.frontPart,
});

const exit = (queue) => {
  if (queue.frontPart === null && queue.backPart === null) return queue; // empty
  let front = queue.frontPart;
  let back = queue.backPart;
  if (front === null) {
    // Reverse backPart into frontPart — the ONLY non-O(1) step, and only when frontPart runs dry
    while (back !== null) {
      front = { value: back.value, next: front };
      back = back.next;
    }
  }
  return { backPart: back, frontPart: front.next };
};
```

Enqueue is always O(1). Dequeue is O(1) *unless* `frontPart` is empty, in which case it's O(n)
to reverse `backPart` — but each element only ever gets moved from back to front once over its
lifetime in the queue. **Amortized cost per operation is still O(1)**: every element is pushed
once, popped once, pushed again (during the reversal), and popped again — four constant-time
steps spread across its lifetime, not four steps *per operation*. This is the same amortized
argument used to justify dynamic array resizing, applied to a purely functional structure.

## Trees: sharing untouched subtrees

The same principle scales to trees: when you insert into a binary search tree functionally,
only the nodes on the path from the root to the insertion point need to be recreated — every
sibling subtree is shared unchanged between the old and new tree:

```js
const insert = (tree, key) => {
  if (tree === null) return { key, left: null, right: null };
  if (key < tree.key) return { ...tree, left: insert(tree.left, key) };
  if (key > tree.key) return { ...tree, right: insert(tree.right, key) };
  return tree; // key already present; share the whole tree unchanged
};
```

For a balanced tree this is O(log n) per update — new nodes only along one root-to-leaf path —
which is why persistent trees (not persistent flat arrays) are the standard backing structure
for large immutable collections (this is what Immutable.js's `Map`/`List` and Clojure's
persistent vectors do under the hood, using wide, shallow tries instead of binary trees for
better constant factors).

## When *not* to bother with structural sharing

- **Small, infrequently-updated objects/arrays**: `{...obj, key: value}` or
  `[...arr, item]` is simpler, and the O(n) copy is irrelevant at small n. Don't build a
  persistent tree to avoid copying a 5-element config object.
- **Write-heavy, read-rarely, single-owner data**: if nothing else ever holds a reference to a
  previous version, there is no benefit to preserving it — just mutate in place and use the
  patterns from `javascript-functional-data-structure-patterns` instead.
- **`Object.freeze`** gives you shallow, runtime-enforced immutability (mutation attempts throw
  in strict mode / silently fail otherwise) but does **not** give you structural sharing or deep
  immutability — nested objects inside a frozen object are still mutable. It's a safety net for
  "did I accidentally mutate this," not a persistent-data-structure implementation technique.

## Common pitfalls

- **Forgetting that `{...obj}` and `[...arr]` are shallow copies.** Nested objects/arrays inside
  are still shared references; mutating `copy.nested.value = x` also mutates the "original."
  Structural sharing is a feature when intentional and a bug when accidental — know which one
  you're getting from a given spread.
- **Rebuilding more of the structure than necessary.** If an update function copies the whole
  array/tree instead of sharing the untouched part, you've paid the O(n) cost of a naive copy
  while writing code that looks like it's being clever about persistence.
- **Assuming amortized O(1) means every single dequeue is fast.** The two-stack queue's
  worst-case single operation is O(n); only the amortized cost over many operations is O(1).
  This matters if you need real-time/predictable per-call latency, not just good average
  throughput.
- **Using `Object.freeze` and believing the whole object graph is now immutable.** It's shallow;
  freeze nested objects individually (recursively) if that's the actual requirement.
- **Choosing a persistent structure when nothing needs the old versions.** If you never keep or
  compare a prior snapshot, structural sharing is pure overhead versus in-place mutation.

## Checklist

- [ ] Identified *why* immutability is needed here (undo/history, shared state without locks,
      React/Redux update semantics, safe concurrent reads) rather than applying it by default
- [ ] Updates only rebuild the path from the root to the change; untouched subtrees/sublists are
      shared, not copied
- [ ] For queue-like access patterns, used the two-stack technique rather than naive tail-append
      on a singly linked list
- [ ] Verified previously-held references remain valid and unchanged after producing a new
      version (the actual definition of "persistent")
- [ ] Didn't reach for structural sharing on small, rarely-updated data where a plain shallow
      copy is simpler and cheap enough
- [ ] `Object.freeze` (if used) is applied at the depth actually required, not assumed to be deep
