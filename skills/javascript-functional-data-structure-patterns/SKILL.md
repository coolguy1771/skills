---
name: javascript-functional-data-structure-patterns
description: Implement core data structures (linked lists, stacks, queues, binary trees, heaps) in idiomatic JavaScript using plain objects, factory functions, closures, and recursion instead of ES6 classes. Use this when the user asks to "implement a linked list/stack/queue/tree in JavaScript", is deciding between classes and closures/factory functions for a data structure, wants a "functional data structure in JS", needs encapsulation without `class` (private state via closures), or has JS code with `this`-binding bugs, accidental mutation, or a class hierarchy that feels heavier than the problem needs. Also load this when reviewing JS data-structure code that mixes array-based and pointer-based (linked) representations and the tradeoffs aren't obvious (why `push`/`pop` are O(1) but `unshift`/`splice` are O(n)).
---

# JavaScript Functional Data Structure Patterns

Most tutorials teach data structures in JS the Java/C++ way: a `class Node` with a
constructor, a `class LinkedList` with methods, `this.head`, `this.next`. That works, but
it's not the only idiomatic option in JavaScript — and often not the best one. JavaScript lets
you build the same structures as **plain objects connected by references, manipulated by pure,
often-recursive functions**, with encapsulation coming from closures rather than `class`
syntax. Recognize which style fits and don't default to classes out of habit.

## The two idiomatic styles, and when each wins

**Class-based (OOP)**: a `class` with private fields (`#head`), methods that mutate `this`.
Good when: the structure needs many stateful methods called repeatedly on the same instance,
consumers expect `instance.method()` ergonomics, or the team's codebase is already
class-heavy for consistency.

**Factory-function / closure-based (functional)**: a plain node is `{ value, next }`; every
operation is a standalone function `(list, ...) => newList` that takes the structure as its
first argument and returns an (often new or mutated-in-place) structure. Good when: you want
each operation testable in total isolation with no hidden instance state, you're composing
operations functionally (`pipe`, `reduce` over a sequence of structure transforms), or you want
the structure to double as an immutable/persistent one later (see
`javascript-persistent-and-immutable-data-structures`) — closures and POJOs make that
transition far easier than retrofitting immutability onto a mutating class.

Neither is "more correct." Arrow functions make the functional style terse enough that it's a
real, first-class option in JS in a way it isn't in most OOP languages:

```js
// Functional style: a node is just data, `next` is a plain reference
const newList = () => null;
const isEmpty = (list) => list === null;
const size = (list) => (isEmpty(list) ? 0 : 1 + size(list.next));

const add = (list, position, value) => {
  if (isEmpty(list) || position === 0) {
    return { value, next: list };          // new head; shares the rest of the old list
  }
  list.next = add(list.next, position - 1, value);
  return list;
};

const at = (list, position) => {
  if (isEmpty(list)) return undefined;
  if (position === 0) return list.value;
  return at(list.next, position - 1);
};
```

There is no `class`, no `this`, no constructor — and therefore no `this`-binding bugs when a
method is passed as a callback (`list.forEach(list.someMethod)` silently breaking `this` is a
classic class-based footgun that doesn't exist here, because these are free functions).

## Encapsulation without `class`: closures and factory functions

When you *do* want to hide internal state (so callers can't reach into `list._head` directly),
use a closure instead of private class fields:

```js
function createStack() {
  let top = null;                 // private: not reachable from outside
  return {
    push(value) { top = { value, next: top }; },
    pop() {
      if (top === null) return undefined;
      const value = top.value;
      top = top.next;
      return value;
    },
    peek: () => top?.value,
    isEmpty: () => top === null,
  };
}

const s = createStack();
s.push(1);
s.push(2);
console.log(s.pop()); // 2
console.log(s.top);   // undefined — truly private, not just underscore-prefixed
```

This is the same trick as an IIFE-based module: the returned object is the only handle to the
enclosed variables. It predates (and doesn't require) ES2022 private class fields (`#top`), and
it composes well when a structure's "instance" is really just the closure's captured scope.

## Array-backed vs. pointer-backed: the tradeoff that catches people off guard

A list (or stack, or queue) can be backed by a JS array *or* by a chain of `{ value, next }`
objects. They are not interchangeable performance-wise, and the array-based version's cost
depends on **which end** you operate on:

| Operation | Array-backed | Pointer-backed (singly linked) |
|---|---|---|
| Push/pop at the **end** | O(1) amortized (`push`/`pop`) | O(n) (must walk to the tail) unless you keep a tail pointer |
| Push/pop at the **front** | O(n) (`unshift`/`shift` — every other element must move) | O(1) (just repoint the head) |
| Access by index | O(1) | O(n) (must walk from the head) |
| Insert/remove in the middle by position | O(n) (`splice` shifts the tail) | O(n) to find the position, O(1) to relink once there |
| Find by value | O(n) (`includes`) | O(n) |

The practical rule: **a stack (LIFO, operates on one end only) is a perfect fit for a plain JS
array using `push`/`pop`** — both are O(1) amortized, and you get no benefit from a hand-rolled
linked implementation. A **queue (FIFO) is the opposite trap**: naively using `array.push()` +
`array.shift()` looks identical to the stack case, but `shift()` is O(n) because every remaining
element must be reindexed. For a real queue, either use a pointer-based list with a maintained
tail pointer (O(1) enqueue and dequeue), or use two arrays/indices to avoid the front-removal
cost entirely (an index cursor into the array instead of physically removing element 0).

```js
// A queue backed by `array.shift()` looks fine and IS an O(n) op per dequeue —
// invisible in small tests, a real bottleneck once the queue holds thousands of items.
const queue = [];
queue.push(1, 2, 3);
queue.shift(); // O(n): every remaining element is copied down one slot

// Pointer-backed queue: O(1) enqueue and dequeue, at the cost of no O(1) random access
const newQueue = () => ({ head: null, tail: null });
const enqueue = (q, value) => {
  const node = { value, next: null };
  if (q.tail) q.tail.next = node; else q.head = node;
  q.tail = node;
  return q;
};
const dequeue = (q) => {
  if (!q.head) return undefined;
  const value = q.head.value;
  q.head = q.head.next;
  if (!q.head) q.tail = null;
  return value;
};
```

## Recursion depth is a real JS constraint

The recursive functional style above is elegant, but JavaScript engines (V8 included) do
**not** guarantee tail-call optimization even for functions written in tail-call form (it's in
the ES2015 spec as an optional feature; V8 has never shipped it). A recursive `size()`,
`find()`, or `add()` over a list with tens of thousands of nodes can throw
`RangeError: Maximum call stack size exceeded`. When a structure might grow large:

- Prefer an iterative (`while`/`for`) version for the hot path, even if it's less elegant, or
- Keep recursion for clarity in code that only ever runs against small/bounded inputs, and
  document that assumption.

```js
// Iterative fallback when list length isn't bounded
const size = (list) => {
  let n = 0;
  for (let node = list; node !== null; node = node.next) n++;
  return n;
};
```

## Building a binary tree / heap the same way

The same object-plus-function pattern extends naturally to trees — a node is
`{ key, left, right }`, and traversal/insertion are recursive functions taking the tree as
their first argument:

```js
const newNode = (key) => ({ key, left: null, right: null });

const insert = (tree, key) => {
  if (tree === null) return newNode(key);
  if (key < tree.key) tree.left = insert(tree.left, key);
  else if (key > tree.key) tree.right = insert(tree.right, key);
  return tree;
};

const preOrder = (tree, visit = (k) => console.log(k)) => {
  if (tree !== null) {
    visit(tree.key);
    preOrder(tree.left, visit);
    preOrder(tree.right, visit);
  }
};
```

A binary **heap**, by contrast, is almost always backed by a plain array (not pointers) because
its shape is always a complete tree — index `i`'s children live at `2i+1` and `2i+2` with no
pointers needed at all:

```js
const parent = (i) => Math.floor((i - 1) / 2);
const left = (i) => 2 * i + 1;
const right = (i) => 2 * i + 2;

const bubbleUp = (heap, i) => {
  while (i > 0 && heap[parent(i)] > heap[i]) {
    [heap[parent(i)], heap[i]] = [heap[i], heap[parent(i)]]; // swap via destructuring
    i = parent(i);
  }
};
```
For *when* to reach for a heap/priority queue at all, see
`priority-queues-segment-trees-and-union-find` — this skill is about the JS implementation
mechanics once you've decided to build one, not the decision to use one.

## Common pitfalls

- **Reaching for a linked-list class when a plain array would do.** If all access is at one
  end (a stack), a JS array with `push`/`pop` is simpler and just as fast — don't hand-roll
  pointers for something the built-in array already does optimally.
- **Building a queue on `array.shift()`** and only discovering the O(n) cost under load.
- **Assuming deep recursion is free.** Functional-style recursive list/tree functions are
  elegant but will blow the call stack on large, unbounded inputs — check expected size before
  choosing recursion over a loop.
- **Passing a class method as a bare callback** (`arr.forEach(myInstance.method)`), losing
  `this` binding — a problem that simply doesn't arise with the closure/factory-function style.
- **Mutating a node that's still referenced elsewhere** when you meant to treat the structure
  as append-only/shared — decide up front whether operations mutate in place or return a new
  structure, and keep that consistent across all functions in the module (mixing the two styles
  in the same data structure is the single most common source of "spooky action at a distance"
  bugs in this style).

## Checklist

- [ ] Chose class vs. closure/factory-function style deliberately, not by default
- [ ] If closures: truly private state (not just `_underscore` convention) for anything callers
      shouldn't touch
- [ ] Matched the backing store (array vs. pointer chain) to the access pattern: one end only
      → array; both ends or O(1) middle insert → pointers with a tracked tail (or two arrays)
- [ ] Confirmed recursion depth is bounded, or provided an iterative fallback for large inputs
- [ ] Consistent mutation policy across all functions operating on the same structure (either
      all mutate in place, or all return a new structure — not a mix)
- [ ] Heap/complete-tree structures use an array with index arithmetic, not pointer nodes
