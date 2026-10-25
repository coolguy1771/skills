---
name: typescript-generics-done-well
description: Write and constrain TypeScript generic functions and generic types correctly — type parameters, constraints (extends), defaults, inference vs. explicit type arguments, generic type aliases, type predicates (`is`), assertion functions (`asserts`), and function overloads. Use when the user asks "how do I make this function/type generic", has a function returning `any` that should preserve its input's type, sees "Argument of type X is not assignable to parameter of type T", is writing a custom type guard (`isFoo`) that isn't narrowing correctly, needs a function that can be called multiple different ways, or is over-genericizing a function/type that doesn't actually need a type parameter. Also load this when reviewing a shared "/utils" helper function to check whether it should be generic and whether its type parameter has the right constraint.
---

# TypeScript Generics Done Well

A generic function or type takes a **type** as an argument the same way a
regular function takes a **value** as an argument — that's the entire
concept. The skill is in constraining type parameters correctly, letting
inference do the work instead of over-annotating, and knowing the handful of
generic-adjacent tools (type predicates, assertion functions, overloads)
that solve problems a plain generic can't.

## Generic functions: parameterize the type, let inference do the rest

```ts
// Before: works, but throws away all type information
const getFirstElement = (arr: any[]) => arr[0];
const first = getFirstElement([1, 2, 3]); // typed `any` — no safety at all

// After: generic — TypeScript infers TMember from the argument
const getFirstElement = <TMember>(arr: TMember[]) => arr[0];
const firstNumber = getFirstElement([1, 2, 3]);   // number | undefined
const firstString = getFirstElement(["a", "b"]);  // string | undefined
```

Terminology, precisely: `TMember` in `<TMember>` is the **type parameter**;
the `number` you'd pass explicitly (`getFirstElement<number>(...)`) is the
**type argument**; a function/type that declares a type parameter is
**generic**. ("Generic" is not a noun for `<T>` itself — the confusion around
that word is most of what makes generics feel harder than they are.)

**Prefer inference over explicit type arguments.** You rarely need to write
`identity<string>("hello")` — TypeScript infers `T` from the runtime
argument. Pass a type argument explicitly only when inference can't recover
it (an empty array literal, a callback with no inferable parameter) or when
you deliberately want to override what would be inferred:

```ts
function identity<T>(arg: T): T { return arg; }
const a = identity(42);          // inferred: number
const b = identity<string>(42);  // Error: 42 not assignable to string — explicit wins over runtime type
```

Unlike a generic *type* (below), a generic *function* doesn't require every
type argument to be supplied — if you omit it, TypeScript infers instead of
erroring.

## Constraints: `extends` narrows what a type parameter can be

An unconstrained `T` supports no operations at all — not `.length`, not
`>`, not property access — because TypeScript can't assume anything about an
arbitrary type. Add a constraint the moment the function body needs a
property or operation to exist:

```ts
// Error: Property 'id' does not exist on type 'T' (no constraint yet)
function getId<T>(item: T) { return item.id; }

// Fixed: constrain T to "at least has an id"
type HasId = { id: number };
function getId<T extends HasId>(item: T) { return item.id; }
```

Constraints on object types are **open, not closed** — `T extends HasId`
means "at least these properties," and passing an object with *more*
properties than `HasId` is fine; you don't get excess-property warnings on
generic constraints the way you would on an object literal assigned directly
to a type.

## Default type parameters

Like default function parameters, `=` gives a type parameter a fallback when
the caller doesn't supply one:

```ts
type ResourceStatus<TContent extends { id: number }, TMetadata extends object = {}> =
  | { status: "available"; content: TContent; metadata: TMetadata }
  | { status: "unavailable"; reason: string };

// TMetadata defaults to {} — no need to pass it when you don't need metadata
type Playlist = ResourceStatus<{ id: number; name: string }>;
```

## Generic types must be fully applied; generic functions don't have to be

This is the sharpest edge between the two:

```ts
type Example = ResourceStatus; // Error: Generic type 'ResourceStatus' requires 2 type argument(s)

function identity<T>(arg: T): T { return arg; }
const result = identity(42);   // fine — T is inferred, no error
```

Generic *types* always require every non-defaulted type argument up front
(there's no runtime value to infer from). Generic *functions* infer missing
type arguments from the call site. Don't expect a generic type alias to
behave like a generic function with optional arguments — give it defaults
(above) if you want that ergonomics.

## Multiple type parameters, and when *not* to add them

Generic types and functions can take as many type parameters as needed, but
each one is a cost: more surface area for the caller to get wrong, more to
read in the signature. Add a type parameter only when the function's
*behavior*, not just its input type, actually depends on it:

```ts
// Over-genericized: T is only ever used as the element type of a fixed-size
// return — a generic here buys nothing the caller couldn't get by just
// annotating the call site, and it makes the signature harder to read.
function makePair<T>(a: T, b: T): [T, T] { return [a, b]; }

// Appropriately generic: TContent and TMetadata vary independently and both
// flow into the shape of what's returned — a real parameterization.
type ResourceStatus<TContent, TMetadata> =
  | { status: "available"; content: TContent; metadata: TMetadata }
  | { status: "unavailable"; reason: string };
```

If a type parameter is used in only one place and never constrains the
relationship between two other things in the signature, ask whether a
concrete type (or a union) would be simpler and just as safe.

## Type predicates: custom narrowing functions

A function returning a plain `boolean` doesn't narrow the caller's type, even
if its logic clearly checks the right shape:

```ts
// Doesn't narrow — return type is inferred as `boolean`
function isAlbum(input: unknown) {
  return typeof input === "object" && input !== null && "title" in input;
}
const run = (x: unknown) => {
  if (isAlbum(x)) x.title; // Error: 'x' is still 'unknown' here
};
```

Add a **type predicate** return type (`input is Album`) to make the check
narrow the argument at every call site:

```ts
interface Album { title: string; artist: string; }

function isAlbum(input: unknown): input is Album {
  return (
    typeof input === "object" && input !== null &&
    "title" in input && "artist" in input
  );
}
const run = (x: unknown) => {
  if (isAlbum(x)) x.title; // OK — x narrowed to Album
};
```

**A type predicate is a promise you're making TypeScript, not something it
verifies.** If the check inside is looser than the predicate claims (e.g. it
checks only `typeof input === "object"` but promises `input is Album`),
TypeScript will not catch the mismatch — treat authoring one with the same
care as `as` or `!`.

## Assertion functions: narrow by throwing instead of branching

An assertion function (`asserts x is T`) narrows the caller's variable for
the rest of the enclosing scope the moment it's called, rather than needing
an `if` branch:

```ts
function assertIsAlbum(input: unknown): asserts input is Album {
  const ok = typeof input === "object" && input !== null &&
    "title" in input && "artist" in input;
  if (!ok) throw new Error("Not an Album!"); // throw when the check FAILS
}

function getAlbumTitle(item: unknown) {
  assertIsAlbum(item);   // throws, or narrows `item` to Album from here on
  console.log(item.title); // Album, no `if` needed
}
```

The same "this is a promise, not a proof" caveat applies — an assertion
function with a check that's looser than what it claims will narrow to a
type that doesn't actually match the runtime value, causing a crash later
instead of at the assertion site. Get the condition's polarity right: assert
functions should throw when the check *fails*, not when it passes.

## Function overloads: different argument counts/shapes, one implementation

Reach for overloads when a function needs genuinely different **call
signatures** (different argument counts, or unrelated argument shapes) that
a single union parameter can't express cleanly:

```ts
function searchMusic(artist: string, genre: string, year: number): void;
function searchMusic(criteria: { artist: string; genre: string; year: number }): void;
function searchMusic(
  artistOrCriteria: string | { artist: string; genre: string; year: number },
  genre?: string,
  year?: number,
): void {
  if (typeof artistOrCriteria === "string") {
    search(artistOrCriteria, genre, year);
  } else {
    search(artistOrCriteria.artist, artistOrCriteria.genre, artistOrCriteria.year);
  }
}

searchMusic("Tame Impala", "Psych Rock", 2015); // matches overload 1
searchMusic({ artist: "Tame Impala", genre: "Psych Rock", year: 2015 }); // matches overload 2
```

**If the argument count is the same across variants and only the type
differs, use a union parameter instead of overloads** — it's less code and
just as safe:

```ts
// Prefer this over two overloads when arity doesn't change:
function searchMusic(query: string | { artist: string; genre: string; year: number }): void {
  if (typeof query === "string") searchByArtist(query);
  else search(query.artist, query.genre, query.year);
}
```

## Common pitfalls

- **Adding a type parameter that's only ever used once and doesn't
  constrain a relationship between two things.** If nothing forces two
  parts of the signature to agree, you likely don't need a generic —
  consider `unknown` or a concrete type instead.
- **Forgetting a constraint and hitting "property does not exist on type
  T."** TypeScript assumes nothing about an unconstrained type parameter;
  every operation the body performs needs a matching `extends` bound.
- **Writing a type predicate or assertion function whose check is looser
  than what it claims.** This compiles fine and fails at runtime, later,
  somewhere confusing — review these as carefully as any `as` cast.
- **Reaching for overloads when a union parameter would do.** Overloads add
  real complexity (separate signatures, one implementation that must satisfy
  all of them) — reserve them for genuinely different argument counts/shapes.
- **Expecting a generic type alias to infer a missing type argument the way
  a generic function does.** Generic types require every non-defaulted
  argument; give them a default (`= {}`) if you want optional-feeling usage.

## Checklist

- [ ] Every type parameter is either constrained (`extends`) to what the
      function body actually needs, or genuinely needs to accept anything
- [ ] Type arguments are inferred at call sites by default; explicit type
      arguments are used only where inference can't or shouldn't apply
- [ ] Every custom type predicate (`is`) and assertion function (`asserts`)
      has a check that actually verifies the full shape it claims to
- [ ] Assertion functions throw on failure, not on success — check the
      condition's polarity
- [ ] Overloads are used only for differing argument counts/shapes; a union
      parameter is used when arity stays the same
- [ ] No type parameter exists solely because "generics feel more correct" —
      each one earns its place by parameterizing real behavior
