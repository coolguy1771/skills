---
name: function-call-overhead-and-parameter-passing
description: Reason about the fixed compiled cost of calling a function — activation-record setup/teardown, register save/restore for leaf vs. non-leaf functions, macro vs. inline vs. real-call expansion, and the cost difference between pass-by-value and pass-by-reference — to decide when a small function should be inlined, why "leaf" functions compile to less code, and which parameter-passing mechanism is cheaper for a given parameter. Use this when the user asks "how much overhead does a function call actually have," "should I make this function inline," "why is a getter/setter this expensive," "is pass-by-value or pass-by-reference faster here," "why does a large struct parameter slow things down," "what makes a function a leaf function," or is reviewing a hot path with many small function calls, large-by-value parameters, or a manual inlining decision. Distinct from call-stack-and-stack-frame-mental-model (the LIFO mental model behind stack-overflow/recursion bugs) and arm64-calling-conventions-and-stack-frames (writing the ABI-compliant assembly by hand) — this is the performance cost model for *deciding*, at the source-language level, when a call is worth its overhead.
---

# Function Call Overhead and Parameter Passing

A function call's overhead is essentially fixed — the same handful of instructions to set
up an activation record, pass parameters, and tear it all down again — regardless of
whether the function body is one instruction or a thousand. That fixed cost is why a
one-line accessor function can be proportionally far more expensive than the "same" code
written inline, and why the parameter-passing mechanism you choose can dominate a small
function's actual runtime cost.

## The fixed overhead of "just" a call

Compare computing `a + b` inline versus through a trivial function:

```c
// Inline: ~3 instructions on a typical CPU
dest = a + b;

// Via a function call: push/move args, call, build+tear down an activation
// record inside the callee, return — roughly 3x the instruction count of the
// inline version, and several of those instructions are memory accesses
// (stack pushes/pops) rather than pure register ops.
int sum(int a, int b) { return a + b; }
...
dest = sum(aParam, bParam);
```

The call version can run several times slower than the inline version for a body this
small — not because summing two numbers is expensive, but because the call/return
machinery costs more than the entire computation it wraps. **The overhead is fixed, so it
matters enormously for small functions and is essentially free to ignore for large ones.**
A function whose body is 200 instructions loses nothing meaningful to ~10 instructions of
call overhead; a function whose body is 3 instructions loses the majority of its runtime
to that same overhead.

## Leaf functions compile to measurably less code

A **leaf function** (one that calls no other function) can skip work a non-leaf function
cannot:

- It never needs to preserve a caller-visible link/return register across a nested call,
  because it never makes one.
- It never needs to save registers purely to protect them across a call it doesn't make.
- On architectures with a dedicated link register (ARM, PowerPC), a leaf function often
  never has to touch memory to save that register at all — the two extra memory accesses
  (save on entry, restore on exit) that a non-leaf function pays are simply absent.

```
; Leaf function (calls nothing else): no link-register save needed.
g:  <body>
    blr                     ; return directly via the link register

; Non-leaf function (calls g twice): must save/restore the link register,
; because calling g overwrites it.
f:  mflr  r0
    stw   r0, 8(r1)         ; save caller's return address
    bl    g
    bl    g
    lwz   r0, 8(r1)
    mtlr  r0                ; restore before returning
    blr
```

**Practical implication**: flattening a call tree (inlining small interior-node functions
into their callers so more of the tree's *leaves* end up directly in bigger functions)
reduces this save/restore overhead throughout the tree — but don't do it by hand at the
expense of your program's modularity and readability. Let macro/inline mechanisms (below)
do this for genuinely small functions instead of manually merging unrelated logic into one
giant function.

## Macros vs. real functions vs. inline functions

- **A pure macro** textually substitutes the body at each call site: no call/return
  overhead, no activation record, parameters substituted directly. The cost is code size —
  every invocation duplicates the full body, which is only worth it for genuinely small
  bodies (rule of thumb: roughly 1-5 statements) called from many sites, or in the rare
  case where speed matters more than code size.
- **An `inline` function/keyword is a *hint*, not a guarantee.** The compiler is free to
  expand it at the call site or compile it as an ordinary out-of-line function with a real
  call — its decision, typically based on body size and parameter count. Even when
  expanded, an inline function *can still need its own activation record* for locals that
  don't fit in registers, unlike a pure macro:

  ```c
  // The compiler DOES expand this one inline (trivial body, no locals):
  inline int inlineFunc(int a, int b) { return a + b; }

  // The compiler may REFUSE to expand this one and emit a real out-of-line
  // call instead — larger body, local array, multiple loops:
  inline int ilf2(int a, int b) {
      int c[4], d;
      for (int m = 0; m < 4; ++m) c[m] = geti();
      d = getj();
      for (int m = 0; m < 4; ++m) d += c[m];
      return (a + d) - b;
  }
  ```

  Never assume the `inline` keyword alone eliminated a call — check the actual compiled
  output (disassembly, `-S`, or a compiler-explorer-style tool) before relying on it for a
  performance-critical path; a function with locals large enough to need real stack
  storage, or a body over the compiler's size threshold, routinely gets compiled as a
  normal call despite being marked `inline`.
- **The classic case this bites you**: a small, pure function sitting inside a loop's
  or `if`'s test condition. If the compiler declines to inline it, every iteration pays a
  full call/return for something as small as a comparison — see
  `hll-conditional-and-boolean-evaluation-costs` for that scenario in detail.

## Parameter passing: register vs. stack, and value vs. reference

- **Register-passed parameters are cheap**; most ABIs pass the first few integer
  parameters in registers specifically because studies of real code show most functions
  take three or fewer parameters. Parameters beyond the register budget spill to the
  stack, which costs a memory write at the call site and a memory read in the callee —
  real overhead per extra parameter once you exceed the register budget.
- **Small scalar parameters are padded to a minimum slot size** (commonly a
  word/double-word) even when the actual type is smaller (a `byte` parameter), purely so
  the stack/registers stay aligned — don't assume a `char` parameter is cheaper to pass
  than an `int` one; on many ABIs they cost the same slot.
- **Pass-by-value**: the caller copies the argument's value into the callee's activation
  record (or a register). Cheap for scalars — often a single push/move. **Expensive for
  large aggregates** (a big struct, a fixed-size array passed by value) because the
  *entire object* must be copied byte-for-byte into the callee's frame on every call —
  avoid passing large structures by value in a hot path.
- **Pass-by-reference**: the caller passes an address (pointer-sized, regardless of the
  referent's size) instead of the value. This makes the *passing* cost constant no matter
  how large the referent is, but makes every *access inside the callee* cost one extra
  dereference — a pointer load, then a load/store through it — versus a single direct
  access for a by-value parameter:

  ```c
  // Pass-by-value access: one instruction to read the parameter.
  // Pass-by-reference access: one instruction to load the pointer,
  // one more to dereference it — every single time the parameter is used.
  void refValue(int *dest, int *passedByRef, int passedByValue) {
      *dest = *passedByRef + passedByValue;
      //      ^^^^^^^^^^^^^   ^^^^^^^^^^^^^
      //      load ptr, deref   direct load
  }
  ```

  **Rule of thumb**: pass small scalars by value, pass large aggregates by reference
  (or `const` reference where the language supports it, to also communicate no
  modification is intended) — the crossover is exactly the point where the one-time copy
  cost of by-value exceeds the per-access dereference cost of by-reference, which depends
  on the object's size and how many times the callee touches it.

## Common pitfalls

- **Writing many tiny, frequently called functions in a hot path and assuming the
  optimizer inlines all of them.** Inlining is a heuristic decision the compiler makes per
  function based on size/complexity — verify with actual compiler output before relying
  on it, especially for anything with local arrays, loops, or multiple statements.
- **Passing large structs or fixed-size arrays by value into a hot-path function.** Every
  call pays a full byte-for-byte copy into the callee's activation record; pass by
  (const) reference instead once the object is bigger than a few machine words.
- **Assuming pass-by-reference is always faster than pass-by-value.** It's cheaper to
  *pass*, but every *access* inside the callee costs an extra dereference — for a small
  scalar accessed many times in a tight loop, by-value can still win.
- **Manually flattening a call tree (inlining by hand) at the expense of readability**,
  when a compiler-respected `inline` hint or letting the optimizer decide would achieve
  the same performance without destroying the code's modular structure.
- **Ignoring that a function's "leaf-ness" affects its own compiled cost.** A function
  that calls nothing else is measurably cheaper to enter/exit than one that does, on
  architectures with an explicit link register — a relevant factor when deciding whether
  to extract one more layer of helper function in a hot path.

## Checklist

- [ ] For a hot path with many small function calls, checked actual compiled output to
      confirm which calls were actually inlined rather than assuming `inline` (or
      compiler heuristics) took effect
- [ ] No large struct/array parameter is passed by value into a frequently called
      function without a specific reason
- [ ] For each reference/pointer parameter, weighed the one-time pass cost against the
      per-access dereference cost given how many times the callee touches it
- [ ] Considered whether a tiny, frequently called pure function belongs in a macro or
      guaranteed-inline form rather than a normal out-of-line call, if profiling shows
      its call overhead matters
- [ ] Didn't manually flatten a call tree at the cost of code clarity when a compiler
      inlining hint would achieve the same effect
- [ ] Checked for a small function sitting inside a loop or `if` condition specifically
      (see `hll-conditional-and-boolean-evaluation-costs`) before assuming its call
      overhead is negligible
