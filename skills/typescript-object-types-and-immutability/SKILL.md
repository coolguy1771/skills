---
name: typescript-object-types-and-immutability
description: Design TypeScript object types (interfaces vs type aliases, intersections vs interface extends, index signatures, Record), and control mutability with readonly, as const, and let/const inference rules. Use when the user asks "should I use type or interface", needs dynamic/unknown object keys, hits "Property does not exist on type '{}'" when building an object incrementally, wants an object or array that can't be mutated, is confused why a const object's property still widens to string when passed to a function, or asks about the difference between as const and Object.freeze. Also load this when reviewing TS object types that mix known and dynamic keys, or reviewing code that needs deep immutability for config objects.
---

# TypeScript Object Types and Immutability

Two related design questions come up constantly in TypeScript codebases:
"how do I shape this object type" and "why won't TypeScript let this stay
narrow/immutable." This skill covers extending and combining object types,
handling dynamic keys, and every layer of TypeScript's mutability model
(`let` vs `const`, `readonly`, `as const`).

## Extending object types: intersections vs. `interface extends`

Two syntaxes combine object types — pick `interface extends` when you can:

```ts
// Intersection: combines any two types, including unions and primitives
type Timestamped = { createdAt: Date };
type Album = { title: string; artist: string };
type TimestampedAlbum = Album & Timestamped;

// interface extends: only works between object shapes, but behaves better
interface Album { title: string; artist: string }
interface StudioAlbum extends Album { studio: string; producer: string }
```

`interface extends` wins for two concrete reasons:

1. **Better errors on incompatible properties.** Intersecting two types with
   a conflicting property silently resolves that property to `never` (a type
   nothing can satisfy) with no error until you try to *use* it. Extending
   an incompatible interface errors immediately, at the declaration:
   ```ts
   type User = { age: number } & { age: string }; // age: never, no error here

   interface User1 { age: number }
   interface User extends User1 { age: string } // Error: incorrectly extends interface 'User1'
   ```
2. **Better TypeScript performance.** An intersection is recomputed every
   time it's referenced; TypeScript caches an interface's resolved shape by
   name and computes it once. On large codebases this measurably affects
   `tsc` and editor responsiveness.

## `type` vs. `interface`

Use `interface` for plain object/function shapes you may need to extend;
reach for `type` when you need something an interface can't express:

- **`type` is more flexible** — it can alias a union, a tuple, a primitive,
  or a mapped/conditional type. `interface` can only describe object and
  function shapes.
- **Interfaces support declaration merging** — two `interface Album {}`
  declarations with the same name in the same scope are automatically merged
  into one. This can be a footgun (an accidental duplicate name silently
  merges instead of erroring) or a deliberate feature (library authors use it
  to let consumers augment a global type). `type` gives a **duplicate
  identifier error** instead, which is usually what you want for
  application code.

There's no universally "correct" default — many teams default to
`interface` for object shapes (to get the two extends benefits above) and
`type` for everything else (unions, tuples, derived/mapped types).

## Dynamic object keys: index signatures, `Record`, and mixing both

TypeScript is conservative about keys it doesn't know about — even code that
looks like normal dynamic JavaScript needs an explicit type to allow it:

```ts
// TypeScript error: can't add unknown keys to `{}`
const awards = {};
awards.Grammy = true; // Error: Property 'Grammy' does not exist on type '{}'
```

**Index signatures** describe "any key of this type maps to this value type":

```ts
const awards: { [key: string]: boolean } = {};
awards.Grammy = true; // OK
```

**`Record<K, V>`** is the more common, more flexible choice — prefer it as
the default for dynamic keys:

```ts
const awards: Record<string, boolean> = {};

// Record supports a union of literal keys; index signatures don't
const nominations: Record<"Grammy" | "MercuryPrize" | "Billboard", boolean> = {
  Grammy: true, MercuryPrize: false, Billboard: true,
};
// The equivalent index signature is a compile error:
// [index: "Grammy" | "MercuryPrize" | "Billboard"]: boolean;
//   Error: An index signature parameter type cannot be a literal type.
```

**Combining known and dynamic keys** — a base set of required keys plus
room for arbitrary extras — is an intersection of `Record` with an index
signature:

```ts
type BaseAwards = "Grammy" | "MercuryPrize" | "Billboard";
type ExtendedAwards = Record<BaseAwards, boolean> & {
  [award: string]: boolean;
};

const nominations: ExtendedAwards = {
  Grammy: true, MercuryPrize: false, Billboard: true,
  "American Music Awards": true, // extra keys still allowed
};
```

## Reducing duplication: `Partial`, `Required`, `Pick`, `Omit`

```ts
interface Album { title: string; artist: string; releaseYear: number; genre?: string }

type DraftAlbum = Partial<Album>;             // every property optional
type FinalAlbum = Required<Album>;            // every property required, incl. genre
type AlbumSummary = Pick<Album, "title" | "artist">;   // only these keys
type AlbumWithoutYear = Omit<Album, "releaseYear">;    // everything except this key
```

`Pick`/`Omit` work on **object types** by key name; they do not work on
unions of object types the way you might expect (a common mix-up — see
`typescript-utility-types-and-type-transformations` for `Exclude`/`Extract`,
their union-shaped counterparts).

## Mutability and inference: `let` vs. `const`

How a variable is declared affects how narrowly TypeScript infers its type,
independent of the object/array mutability rules below:

```ts
type Genre = "rock" | "country" | "electronic";
const accepts = (genre: Genre) => {};

let choice = "rock";
accepts(choice); // Error: 'string' is not assignable to parameter of type 'Genre'
//               // widened to `string` because `let` could reassign it later

const choice2 = "rock";
accepts(choice2); // OK — `const` can never change, so TS infers the literal "rock"
```

**Object properties widen the same way, even inside a `const` object** — the
`const` only guarantees the *variable binding* won't be reassigned, not that
the object's properties can't be:

```ts
type Attrs = { status: "new-release" | "on-sale" | "staff-pick" };
const update = (attrs: Attrs) => {};

const attrs = { status: "on-sale" };
update(attrs); // Error: property 'status' inferred as `string`, not the literal

update({ status: "on-sale" });          // OK — inlined, so it can't be reassigned first
const attrs2: Attrs = { status: "on-sale" }; // OK — annotation pins the literal type
```

## `readonly` properties and arrays

`readonly` is a compile-time-only lock on a specific property or array —
JavaScript still allows the mutation at runtime; TypeScript just refuses to
compile code that attempts it:

```ts
interface Album {
  readonly title: string;
  readonly artist: string;
  status?: "new-release" | "on-sale"; // still mutable
}

function printNames(names: readonly string[]) {
  names.push("new"); // Error: Property 'push' does not exist on type 'readonly string[]'
}
// same thing, spelled as a generic: ReadonlyArray<string>
```

`Readonly<T>` wraps every property of an object type at once:

```ts
const frozen: Readonly<Album> = { title: "OK Computer", artist: "Radiohead" };
frozen.title = "..."; // Error: Cannot assign to 'title' because it is a read-only property
```

`Readonly<T>` and the `readonly` modifier only go **one level deep** — nested
objects and arrays inside a readonly-marked property remain fully mutable
unless you mark them too.

## `as const`: deep immutability and literal inference

`as const` marks a value and *everything nested inside it* as `readonly`,
and infers every property using its most specific literal type instead of
the general one:

```ts
const attrs = { status: "on-sale" } as const;
// inferred as: { readonly status: "on-sale" }  (not `string`)

attrs.status = "new-release"; // Error: Cannot assign to 'status' — readonly
```

This is ideal for large config objects and for deriving literal unions from
an array (see `typescript-utility-types-and-type-transformations` for the
`(typeof arr)[number]` pattern this enables).

**`as const` loses to a variable annotation, always.** If you combine them,
the annotation wins and the value's own type is discarded — a subtle trap:

```ts
type Attrs = { status: "new-release" | "on-sale" | "staff-pick" };
const attrs: Attrs = { status: "on-sale" } as const;
attrs.status = "new-release"; // No error — the `as const` readonly-ness is gone!
```

**`as const` vs. `Object.freeze`**: `Object.freeze` is a *runtime* guarantee
(it actually throws/no-ops on mutation in strict mode) but its TypeScript
type — `Readonly<T>` — is only **one level deep**, exactly like the
`readonly` modifier above. `as const` is compile-time only (no runtime
protection at all) but is **deeply** readonly through the whole nested
structure. Use `as const` for type-level guarantees on nested config data;
use `Object.freeze` when you also need the runtime enforcement, and combine
both if you need both guarantees.

## Common pitfalls

- **Trying to add a key to an object typed as `{}`** instead of using
  `Record<string, T>` or an index signature from the start.
- **Using an index signature when you have a known, finite set of literal
  keys.** Index signatures reject literal-type keys entirely — use `Record`.
- **Expecting `Pick`/`Omit` to work on a union of object types.** They only
  operate on a single object type's keys; use `Extract`/`Exclude` for unions.
- **Assuming a `const` object is deeply immutable.** `const` only locks the
  variable binding; properties (and especially nested objects/arrays) stay
  mutable — and TypeScript will still widen their inferred types — unless
  you add `readonly`, `Readonly<T>`, or `as const`.
- **Combining `as const` with a variable type annotation** and being
  surprised the object is mutable again — the annotation always overrides
  the assertion.
- **Assuming `Readonly<T>` or `as const` gives runtime immutability.** Both
  are compile-time only; use `Object.freeze` if you need the object to
  actually throw/no-op on mutation at runtime.

## Checklist

- [ ] Object shapes that may be extended use `interface`; unions, tuples, and
      derived types use `type`
- [ ] Dynamic-key objects use `Record<K, V>` (not a bare index signature)
      unless the keys are truly unbounded strings
- [ ] Fixed-shape config objects that shouldn't change use `as const`, not
      just a `const` declaration
- [ ] No variable annotation is combined with `as const` on the same value
      (the annotation silently wins and drops the assertion)
- [ ] Readonly guarantees that need to hold at runtime use `Object.freeze`,
      not just `readonly`/`as const`, which are compile-time only
- [ ] Values passed into functions expecting a literal-typed field are
      either inlined or explicitly annotated, not left to a bare `const`
      object whose property types would otherwise widen
