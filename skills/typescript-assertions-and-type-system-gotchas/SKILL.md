---
name: typescript-assertions-and-type-system-gotchas
description: Choose correctly between variable annotations, `satisfies`, `as`, and the non-null assertion (`!`) when TypeScript needs to be told something it can't infer, know when to reach for `@ts-expect-error` vs `as any`, and recognize TypeScript's "weird parts" — excess property checks that only fire on fresh object literals, the empty object type `{}` meaning "not null/undefined", evolving `any`, and how unions of functions intersect their parameters. Use when the user asks "should I use `as` or `satisfies`", is fighting a false-negative where an object with a typo'd property isn't flagged, sees "Object literal may only specify known properties" on one line but not another with the same object, is deciding whether to suppress a TypeScript error, or is confused why passing more/fewer arguments to a callback type behaves asymmetrically. Also load this when reviewing a PR that adds `as any`, `@ts-ignore`, or a non-null assertion, to check whether a safer alternative applies.
---

# TypeScript Assertions and Type System Gotchas

TypeScript sometimes needs information you have and it doesn't ("I know this
regex match isn't null"), and sometimes behaves in ways that look like bugs
but are deliberate, documented trade-offs (excess property checks that
vanish depending on how you write the object). This skill covers both:
picking the right tool to tell TypeScript something, and recognizing the
type system's sharp corners before they cost you a debugging session.

## Variable annotations vs. `satisfies`: who "wins"

Annotating a **variable** makes the annotation win over the value's own
inferred shape — useful for validation, costly for narrow access:

```ts
type Color = string | { r: number; g: number; b: number };
const config: Record<string, Color> = {
  foreground: { r: 255, g: 255, b: 255 },
  border: "transparent",
};
config.foreground.r; // Error: Property 'r' does not exist on type 'Color'
//                   // TS forgot the value's real shape once the annotation applied
```

Dropping the annotation lets the *value* win — you get narrow access back,
but lose the validation that catches a malformed `config` at the definition
site.

**`satisfies` gives you both**: it validates the value against a type
without discarding the value's own inferred (narrower) shape:

```ts
const config = {
  foreground: { r: 255, g: 255, b: 255 },
  border: "transparent",
} satisfies Record<string, Color>;

config.foreground.r;          // OK — inferred as the literal object shape
config.border.toUpperCase();  // OK — inferred as string
config.somethingNew = "red";  // Error — satisfies still locks the key set, unlike no annotation at all
```

Rule of thumb: annotate the variable when you want the wider declared type
to be what callers see; use `satisfies` when you want validation *plus* the
narrowest possible inferred type; use no annotation at all only when you
don't need validation.

## `as`: forcing a type, with limits

`as` overrides TypeScript's inferred type for an expression. It only works
between types with "sufficient overlap" — `as` won't convert between
completely unrelated types unless you go through `unknown` first, which
should read as a warning sign, not a workaround to reach for casually:

```ts
const id = new URLSearchParams(location.search).get("id") as string;
// Unsafe: id is `null` at runtime if the param is missing — as does nothing at runtime

const sales = "Heroes" as number;
// Error: neither type sufficiently overlaps — TS is trying to stop a likely mistake

const sales2 = "Heroes" as unknown as number; // "works" — and is exactly as dangerous as `as any`
```

Prefer a real narrowing check (`typescript-narrowing-and-discriminated-unions`)
over `as` wherever a runtime check is actually possible — `as` performs zero
runtime work, so a wrong assertion becomes a runtime crash instead of a
compile error.

## The non-null assertion (`!`)

`x!` tells TypeScript "trust me, this isn't `null`/`undefined`" — it's `as`
scoped specifically to nullability, and just as unchecked at runtime:

```ts
const id = searchParams.get("id")!;      // crashes at `.toUpperCase()` if actually null
user.profile!.bio;                        // same risk, one property deep
```

It's easy to miss in review because it's a single character. Prefer optional
chaining (`user.profile?.bio`) when a possibly-missing value should
gracefully resolve to `undefined` instead of crashing; reserve `!` for cases
where you've already established the value can't be missing through logic
TypeScript just can't see (and consider a comment explaining why).

## Suppressing errors: prefer `@ts-expect-error` and `as any`, avoid `@ts-ignore`/`@ts-nocheck`

```ts
// @ts-expect-error — errors if this line does NOT produce an error (self-cleaning)
const result = addOne("one");

// @ts-ignore — silently no-ops even once the underlying error is fixed (silent rot)
// @ts-ignore
const result2 = addOne("one");

// @ts-nocheck at the top of a file disables type checking for the WHOLE file
```

`@ts-expect-error` is strictly better than `@ts-ignore` for known,
intentional errors (e.g. a test asserting bad input is rejected) because it
itself errors once the suppressed error goes away — it can't silently rot.
Avoid `@ts-nocheck` entirely in application code; it's easy to forget it's
there and lose type checking for an entire file.

When you need to suppress an error on a *value* rather than a whole line
(passing a deliberately-wrong value into a function, say), `as any` is often
preferable to a directive comment: it scopes the unsafety to one expression
instead of the entire line, which matters when a line has more than one
thing that could be wrong.

## The evolving `any`

A `let` declared with no initial value and no annotation starts as `any`,
but narrows to whatever's actually assigned each time — closer to
progressive narrowing than a true, permanent `any`:

```ts
let value;              // any
value = 123;
value.toExponential();  // OK — narrowed to number
value = "text";
value.toUpperCase();    // OK — narrowed to string
value = 123;
value.toUpperCase();    // Error: Property 'toUpperCase' does not exist on type 'number'
```

This also applies to arrays built up with `.push` from an empty literal
(`const arr = []` starts as `any[]`, then narrows to the union of pushed
element types). Useful to recognize so you don't mistake it for a real,
static `any` that would allow calling anything at any point.

## Excess property checks: only on "fresh" object literals

TypeScript checks for unknown/typo'd properties **only when an object
literal is written directly at the assignment or call site** ("fresh") —
not when it goes through a variable first ("stale"):

```ts
interface Album { title: string; releaseYear: number; }
const process = (album: Album) => {};

process({ title: "x", releaseYear: 1965, label: "Parlophone" });
//                                         ^^^^^ Error: excess property 'label'

const rubberSoul = { title: "x", releaseYear: 1965, label: "Parlophone" };
process(rubberSoul); // No error — the excess property check doesn't run on a variable
```

This is deliberate: assigning through a variable might reuse that object
elsewhere with legitimately-extra properties, so TypeScript only checks
fresh literals. The practical cost: **a typo'd optional property silently
does nothing** if it's assigned to a variable first —

```ts
const fetch = (opts: { url: string; timeout?: number }) => {};
const options = { url: "/", timeOut: 1000 };  // typo: capital O
fetch(options); // No error — `timeOut` is just a silently-ignored excess property
```

The same leniency applies **when comparing two function types** (e.g.
passing a callback whose return value has an extra property) — TypeScript
only checks excess properties when directly comparing an object literal
against a type, not when comparing one function's inferred return type
against another. Add an explicit return type annotation to the callback to
force the check.

## The empty object type `{}` means "not null or undefined"

`{}` is not "an object with no properties" — it's **the type of anything
that isn't `null` or `undefined`**, because every non-nullish JS value
(strings, numbers, booleans, functions, objects) has properties:

```ts
const a: {} = "a string";      // OK
const b: {} = 42;              // OK
const c: {} = { anything: 1 }; // OK
const d: {} = null;            // Error
const e: {} = undefined;       // Error
```

If you actually want "an object with no required properties, structurally,"
that's `Record<string, unknown>` or a specific shape — not `{}`. Recognize
`{}` in a hover/error message as "not nullish," not as "empty object."

## Iterating object keys: `Object.keys` returns `string[]`, not `keyof`

```ts
const album = { title: "x", artist: "y" };
Object.keys(album).forEach((key) => {
  album[key]; // Error: string can't be used to index this object
});
```

Because objects are open in TypeScript (more properties might exist at
runtime than the type declares), `Object.keys`/`for...in` can only promise
`string`, not the specific key union. Cast deliberately
(`key as keyof typeof album`) only when you're certain no extra keys exist
on the actual runtime value — this is a real safety trade-off, not a free
fix, since a genuinely wider runtime object will throw at the cast-open
property access.

## Function assignability: fewer params is fine, more params is not

TypeScript allows a callback to declare **fewer** parameters than it'll
actually be called with (exactly like `array.map((x) => x)` ignoring the
index/array arguments) but not **more**:

```ts
type Callback = (filename: string, volume: number, bassBoost: boolean) => void;
declare function handlePlayer(cb: Callback): void;

handlePlayer((filename) => {});                    // OK — ignores extra args
handlePlayer((filename, volume, bassBoost, extra) => {}); // Error — too many params expected
```

Type the callback parameter with the **maximum** argument count you'll ever
call it with; callers are free to accept fewer.

**Unions of function types intersect their parameters**, not union them —
this trips people up because it's the opposite of what union syntax
suggests:

```ts
const formatters = {
  title: (album: { title: string }) => album.title,
  artist: (album: { artist: string }) => album.artist,
};
// typeof formatters[key] is a UNION of the two function types, but calling
// it requires an argument satisfying the INTERSECTION of their parameters:
type Fn = (typeof formatters)[keyof typeof formatters];
// (album: { title: string } & { artist: string}) => string
```

This is correct once you think it through — a value typed as "one of these
functions, I don't know which" must be called safely for *either* one, so
the argument has to satisfy both simultaneously. Design APIs so callers
don't have to reason about this directly (e.g. look the function up by key
and call it before the type collapses to a union).

## Common pitfalls

- **Annotating a variable when you actually wanted `satisfies`.** If you
  find yourself losing narrow property access right after adding a type
  annotation for validation, that's the signal to switch to `satisfies`.
- **Using `as any` or `!` where a real narrowing check or optional chain
  would work.** Both compile away to nothing at runtime — a wrong one
  becomes a production crash, not a caught bug.
- **Trusting that an object assigned to a variable will be checked for
  excess/typo'd properties the way an inline literal would.** It won't —
  annotate the variable explicitly if you want the check.
- **Reading `{}` as "empty object" in an error message** instead of "not
  null or undefined" — this misreading wastes time chasing the wrong bug.
- **Casting `Object.keys(obj)` results to `keyof typeof obj`** without being
  sure no extra runtime keys exist — this reintroduces exactly the runtime
  error TypeScript's `string[]` typing was protecting against.

## Checklist

- [ ] Values that need both validation and narrow inferred access use
      `satisfies`, not a variable annotation
- [ ] Every `as`, `!`, and `@ts-ignore`/`@ts-nocheck` in the diff has a
      justification a reviewer could not replace with a real narrowing check
- [ ] `@ts-expect-error` is preferred over `@ts-ignore` for intentionally
      broken lines (e.g. in tests), since it self-invalidates when fixed
- [ ] No `@ts-nocheck` in application code
- [ ] Optional properties on objects passed through variables (not inline)
      are double-checked for typos, since excess-property checking won't
      catch them
- [ ] Callback parameter types declare the maximum arguments the callback
      will ever receive, not the minimum a particular caller happens to use
