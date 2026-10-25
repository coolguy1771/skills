---
name: typescript-utility-types-and-type-transformations
description: Derive one TypeScript type from another instead of duplicating it, using keyof, typeof, indexed access types, Parameters/ReturnType/Awaited, Exclude/Extract/NonNullable, mapped types (including key remapping with `as`), conditional types, and template literal types. Use when the user wants a type that stays in sync with an object/function/array instead of being hand-maintained, asks about keyof or typeof in TypeScript, needs to extract a function's argument or return type, wants a union of all values in an object or array, is writing a mapped type like `{ [K in keyof T]: ... }`, needs conditional (`T extends U ? X : Y`) or template literal string types, or has two types that are manually kept in sync and drift out of sync over time. Also load this when reviewing TypeScript code with duplicated type definitions that could be derived from a single source of truth.
---

# TypeScript Utility Types and Type Transformations

Whenever a type shape already exists somewhere in your code — an object
literal, a function, an array of literals — deriving a new type *from* it
keeps them in sync automatically. Manually re-declaring `"development" |
"production" | "staging"` next to a `configurations` object that already has
those keys is a bug waiting for the next time someone adds an environment
and forgets the type. This skill covers TypeScript's toolkit for deriving,
transforming, and pattern-matching over types.

## `keyof` and `typeof`: the two building blocks

`keyof` turns an object type's keys into a union of string literals.
`typeof` (in type position) turns a *value* into the type describing it —
they're almost always used together, because most values worth deriving
keys from are objects, not object *types*:

```ts
interface Album { title: string; artist: string; releaseYear: number; }
type AlbumKeys = keyof Album; // "title" | "artist" | "releaseYear"

const configurations = {
  development: { apiBaseUrl: "http://localhost:8080", timeout: 5000 },
  production: { apiBaseUrl: "https://api.example.com", timeout: 10000 },
};
// Derived — adding a new key to `configurations` automatically updates this:
type Environment = keyof typeof configurations; // "development" | "production"
```

`typeof` only works on values in scope; you can't create a runtime value
from a type (there's no inverse operation) — the direction only goes
value → type.

## Indexed access types: reach into a type by key

```ts
type ApiBaseUrl = (typeof configurations)["development"]["apiBaseUrl"]; // string

// Passing a union of keys returns a union of the corresponding value types:
type Env = "development" | "production";
type SomeConfig = (typeof configurations)[Env];
```

**`[number]` on an array or tuple type extracts the union of every element's
type** — the standard way to turn an `as const` array into a literal union,
and the reason `as const` arrays are usually a better source of truth than a
hand-written union:

```ts
const programModes = ["group", "1on1", "selfDirected"] as const;
type ProgramMode = (typeof programModes)[number]; // "group" | "1on1" | "selfDirected"
// New items pushed onto the array source flow into the type automatically —
// no manual union to keep in sync.
```

(`keyof typeof programModes` does *not* give you this — for an array it
includes every array method/index as a key too. Use `[number]`, not `keyof`,
for arrays.)

## Deriving from functions: `Parameters`, `ReturnType`, `Awaited`

```ts
function sellAlbum(album: Album, price: number, quantity: number) { return price * quantity; }

type SellAlbumParams = Parameters<typeof sellAlbum>; // [Album, number, number]
type Price = SellAlbumParams[1];                     // number — index into the tuple
type SellAlbumReturn = ReturnType<typeof sellAlbum>;  // number
```

Always wrap a function value in `typeof` before passing it to `Parameters`
or `ReturnType` — passing the function directly is a category error
TypeScript flags immediately (`'x' refers to a value, but is being used as a
type here. Did you mean 'typeof x'?`).

`ReturnType` on an `async` function gives you the `Promise`-wrapped type, not
the resolved value — unwrap it with `Awaited`:

```ts
const fetchUser = async (id: string) => ({ id, name: "John Doe" });
type UserPromise = ReturnType<typeof fetchUser>; // Promise<{ id: string; name: string }>
type User = Awaited<ReturnType<typeof fetchUser>>; // { id: string; name: string }
```

This trio (`Parameters`/`ReturnType`/`Awaited`) is the standard way to type
against a function you don't own the implementation of — a third-party
library, a generated client, another team's code.

## Set-style transforms: `Exclude`, `Extract`, `NonNullable`

These operate on **unions**, by pattern-matching each member against a
pattern — don't confuse them with `Omit`/`Pick`, which operate on a single
object type's **keys** (see `typescript-object-types-and-immutability`):

```ts
type AlbumState =
  | { type: "recording"; studio: string }
  | { type: "mixing"; engineer: string }
  | { type: "released"; releaseDate: string };

type UnreleasedState = Exclude<AlbumState, { type: "released" }>; // recording | mixing
type RecordingState = Extract<AlbumState, { type: "recording" }>; // just the recording member

type Example = "a" | "b" | 1 | 2;
type OnlyNumbers = Exclude<Example, string>; // 1 | 2
type OnlyStrings = Extract<Example, string>; // "a" | "b"
```

`NonNullable<T>` is `Exclude<T, null | undefined>` spelled more clearly —
prefer it when that's specifically what you're doing:

```ts
type Album = { artist?: { name: string } };
type Artist = NonNullable<Album["artist"]>; // { name: string } — no undefined
```

| Name | Operates on | Effect |
|---|---|---|
| `Exclude<U, M>` | Unions | Removes matching union members |
| `Extract<U, M>` | Unions | Keeps only matching union members |
| `Omit<T, K>` | Object types | Removes named keys |
| `Pick<T, K>` | Object types | Keeps only named keys |

## Conditional types: if/else for types

`T extends U ? X : Y` checks whether `T` is assignable to `U`:

```ts
type ToArray<T> = T extends any[] ? T : T[];

type A = ToArray<string>;   // string[]
type B = ToArray<string[]>; // string[] — not string[][], thanks to the check
```

Read `T extends U` as "could a value of type `T` be passed where `U` is
expected" — the same question `extends` asks in a generic constraint.
Conditional types are what makes utility types like `Exclude`/`NonNullable`
possible under the hood; you'll write your own rarely in application code,
but recognize the pattern when you see it in library type definitions.

## Mapped types: build an object type by iterating another's keys

```ts
interface Album { name: string; artist: string; songs: string[] }

type Nullable<T> = {
  [K in keyof T]?: T[K] | null;
};
type AlbumWithNullable = Nullable<Album>;
// { name?: string | null; artist?: string | null; songs?: string[] | null }
```

`[K in keyof T]` iterates every key of `T`; `T[K]` (an indexed access,
above) fetches that key's original type; `?` and `| null`/`readonly` layer
on the same modifiers you'd write on a normal object type.

**Key remapping with `as`** changes the key itself while keeping the
original key available to index with:

```ts
type Uppercased<T> = {
  [K in keyof T as Uppercase<K & string>]: T[K];
};
type AlbumUpper = Uppercased<Album>; // { NAME: string; ARTIST: string; SONGS: string[] }
```

Without the `as` remap, `[K in Uppercase<keyof Album>]: Album[K]` fails —
once `K` itself is transformed, it can no longer be used to index the
original type. `as` keeps a private, untransformed copy of `K` for indexing
while emitting the transformed version as the actual key.

Mapped types also iterate a plain union of string literals, not just
`keyof` of an object — useful for generating a lookup object from a union.

## Template literal types: string patterns as types

```ts
type ImageFile = `${string}.png` | `${string}.jpg`;
const cover: ImageFile = "cover.png"; // OK
const bad: ImageFile = "cover.txt";   // Error

// Combined with a union, produces the cross product of every combination:
type Environment = "dev" | "prod";
type Level = "info" | "error";
type LogPrefix = `[${Environment}] [${Level}]`; // 4 combinations
```

These are the type-level equivalent of JS template literals — useful for
enforcing string formats (routes, CSS units, event names) at the type level
without runtime validation.

## Deriving vs. decoupling: when *not* to derive

Deriving a type couples it to its source — if `AvatarProps` is
`Pick<User, "name" | "imageUrl">`, moving or splitting `User` later forces
you to revisit every type derived from it. For small, narrowly-scoped types
(especially component prop types), it's often better to **decouple**: write
the small type by hand, independent of the larger source type, even though
it duplicates a couple of field names:

```ts
// Derived — coupled to User's exact shape and its module location
type AvatarProps = Pick<User, "name" | "imageUrl">;

// Decoupled — a little duplication, but free of User entirely
type AvatarProps = { name: string; imageUrl: string };
```

Derive when the source is a true single source of truth you want changes to
propagate from automatically (config objects, API response shapes, function
signatures you don't own). Decouple when the consumer's needs are
independent of the source's evolution and coupling would create fragile,
hard-to-trace dependency chains.

## Common pitfalls

- **Passing a value instead of `typeof value`** to `keyof`, `Parameters`,
  or `ReturnType` — these operate on types, not values; the error message
  ("refers to a value, but is being used as a type here") always suggests
  the `typeof` fix.
- **Using `keyof typeof arr` on an array** expecting a union of its values —
  it returns array indices and method names, not element values. Use
  `(typeof arr)[number]` instead.
- **Confusing `Exclude`/`Extract` (unions) with `Omit`/`Pick` (object
  keys).** Trying to `Pick` from a union, or `Omit` a property from a union
  of objects, doesn't do what you want — see the pattern where you need it
  and pick the row from the table above.
- **Forgetting `as` when remapping mapped-type keys**, and getting "Type 'K'
  cannot be used to index type 'T'" once `K` has already been transformed.
- **Deriving deeply nested chains of types from types from types** until a
  small source change ripples through the entire codebase — decouple
  leaf-level types (especially component props) instead of deriving
  everything from one shared root type.

## Checklist

- [ ] Any type that duplicates the shape of an existing object/function/array
      is derived from it (`keyof typeof`, `Parameters`, `ReturnType`,
      `(typeof arr)[number]`) instead of hand-maintained separately
- [ ] `typeof` wraps every value passed to a type-level operator (`keyof`,
      `Parameters<>`, `ReturnType<>`)
- [ ] Array-derived literal unions use `(typeof arr)[number]`, not
      `keyof typeof arr`
- [ ] Union-shaped filtering uses `Exclude`/`Extract`; object-key filtering
      uses `Omit`/`Pick` — not swapped
- [ ] Mapped types that rename keys use `[K in keyof T as ...]`, not a bare
      transform of `K` in the iteration position
- [ ] Small, leaf-level types (especially UI component props) are decoupled
      by hand rather than derived through a long chain from a large shared
      source type
