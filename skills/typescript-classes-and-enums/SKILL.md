---
name: typescript-classes-and-enums
description: Write TypeScript classes with correct access modifiers, safe inheritance (override, abstract classes/methods, implements), and constructor parameter properties, plus decide when (not) to use TypeScript-only runtime features like enums and namespaces. Use when the user is designing a class hierarchy in TypeScript, asks about public/private/protected, wants to enforce a class implements an interface, hits "Cannot create an instance of an abstract class", is deciding between an enum and a union type / `as const` object, asks "should I use enums in TypeScript", or is confused by enum behavior like reverse mappings or Object.keys returning extra entries. Also load this when reviewing a TypeScript class hierarchy for a missing `override` keyword or an enum that could be a plain union type.
---

# TypeScript Classes and TypeScript-Only Runtime Features

TypeScript classes are mostly "JavaScript classes plus a type layer," but
that type layer adds real safety (access modifiers, `implements`, `override`)
that plain JavaScript can't express. Separately, TypeScript ships a few
*runtime* features — enums, namespaces, parameter properties — that don't
exist in JavaScript at all and compile to non-obvious output. Know both:
how to build a correct class hierarchy, and when a TypeScript-only feature
is worth its transpilation cost.

## Constructors, inference, and parameter properties

TypeScript infers constructor-assigned property types, so explicit
annotations on class fields are documentation, not a requirement:

```ts
class Album {
  title: string;
  artist: string;
  releaseYear: number;

  constructor(opts: { title: string; artist: string; releaseYear: number }) {
    this.title = opts.title;
    this.artist = opts.artist;
    this.releaseYear = opts.releaseYear;
  }
}
```

**Parameter properties** (a TypeScript-only feature) collapse "declare the
field, assign it in the constructor" into the constructor's parameter list —
adding a visibility modifier before a parameter both declares the field
*and* assigns it:

```ts
class Rating {
  constructor(public value: number, private max: number) {}
}
// Equivalent JavaScript output does the assignment for you:
// class Rating {
//   constructor(value, max) { this.value = value; this.max = max; }
// }
```

This is real time savings on data-holder classes, but the generated
JavaScript doesn't visually resemble the TypeScript source — worth knowing
before you're debugging a stack trace or reading transpiled output.

## Access modifiers: `public`, `private`, `protected`, `readonly`

```ts
class Album {
  protected title = "";     // accessible in this class and subclasses, not outside
  private catalogId = "";   // accessible only inside this exact class
  readonly artist: string;  // compile-time-only: can't be reassigned after construction
  constructor(artist: string) { this.artist = artist; }
}
```

`private`/`protected` are compile-time only (erased at runtime, unlike
JavaScript's native `#private` fields) — don't rely on them for real
encapsulation against determined callers; use `#field` syntax if you need
actual runtime privacy.

## Inheritance: `extends`, `super`, and safe overrides

```ts
class SpecialEditionAlbum extends Album {
  bonusTracks: string[];
  constructor(opts: { title: string; artist: string; releaseYear: number; bonusTracks: string[] }) {
    super(opts);                    // must run before using `this`
    this.bonusTracks = opts.bonusTracks;
  }
}
```

Overriding a parent method is easy to get silently wrong: if the parent's
method signature changes later, a subclass method that happened to share its
old name just stops overriding it — no error, just a bug. The `override`
keyword makes that a compile error instead:

```ts
class SpecialEditionAlbum extends Album {
  override displayInfo() {           // errors if Album.displayInfo is renamed/removed
    super.displayInfo();
    console.log(`Bonus tracks: ${this.bonusTracks.join(", ")}`);
  }
}
```

Turn on `"noImplicitOverride": true` in `tsconfig.json` to *require*
`override` on every method that overrides a base class method — this catches
the mistake even before you rename anything, by forcing every override to be
declared as intentional up front.

## `implements`: enforcing a shape without inheriting behavior

```ts
interface IAlbum {
  title: string; artist: string; releaseYear: number; trackList: string[];
}

class Album implements IAlbum {
  // Error: Property 'trackList' is missing — implements checks the class
  // literally has every member, the way it would check an object literal
  title: string; artist: string; releaseYear: number;
  constructor(opts: { title: string; artist: string; releaseYear: number }) {
    this.title = opts.title; this.artist = opts.artist; this.releaseYear = opts.releaseYear;
  }
}
```

`implements` only checks the shape — it adds no runtime behavior and doesn't
let the class use the interface's own default logic (interfaces have none).
Use it to enforce a contract across otherwise-unrelated classes; use
`extends` when you want to actually share and reuse implementation.

## Abstract classes: partway between an interface and a base class

```ts
abstract class AlbumBase {
  title: string; trackList: string[] = [];
  constructor(opts: { title: string }) { this.title = opts.title; }

  addTrack(track: string) { this.trackList.push(track); } // shared, concrete
  abstract addReview(author: string, review: string): void; // must be implemented by subclasses
}

new AlbumBase({ title: "x" }); // Error: Cannot create an instance of an abstract class

class Album extends AlbumBase {
  addReview(author: string, review: string) { /* ... */ } // required by `abstract`
}
```

Reach for an abstract class instead of a plain interface when subclasses
should get real shared implementation (like `addTrack` above) in addition to
a required contract (`addReview`) — an interface alone can't provide the
former.

## Enums: what they buy you, and their sharp edges

`enum` is one of the TypeScript-only *runtime* features — it compiles to
real JavaScript, unlike types/interfaces which vanish entirely.

```ts
enum AlbumStatus { NewRelease, OnSale, StaffPick } // numeric, auto-incremented 0,1,2
enum AlbumStatus2 { NewRelease = "NEW_RELEASE", OnSale = "ON_SALE" } // string enum
```

**Numeric enums generate a reverse mapping**, which is usually a surprise:

```ts
// AlbumStatus above transpiles to roughly:
var AlbumStatus = { 0: "NewRelease", 1: "OnSale", 2: "StaffPick", NewRelease: 0, OnSale: 1, StaffPick: 2 };
Object.keys(AlbumStatus); // ["0", "1", "2", "NewRelease", "OnSale", "StaffPick"] — not what you'd expect
```

String enums don't get a reverse mapping (cleaner `Object.keys` output), but
in exchange they're the **one type in TypeScript compared nominally instead
of structurally** — two string enums with identical members but different
names are incompatible types, unlike every other TS type:

```ts
enum StatusA { X = "x" }
enum StatusB { X = "x" }
function take(s: StatusA) {}
take(StatusB.X); // Error — even though the runtime values are identical
```

Numeric enums, confusingly, don't have this restriction — any number is
assignable to a numeric enum type, valid member or not, unless it's a
literal the compiler can check.

## Should you use enums? Prefer `as const` objects

For a new project, prefer a plain object with `as const` (or a union of
string literals) over `enum` — it has none of the enum quirks above, adds no
runtime-representation surprises, and is a plain, structurally-typed
TypeScript value:

```ts
const AlbumStatus = {
  NewRelease: "NEW_RELEASE",
  OnSale: "ON_SALE",
  StaffPick: "STAFF_PICK",
} as const;
type AlbumStatus = (typeof AlbumStatus)[keyof typeof AlbumStatus];
// "NEW_RELEASE" | "ON_SALE" | "STAFF_PICK" — structurally typed, no reverse mapping
```

The trade-off: this requires passing the actual string value
(`AlbumStatus.OnSale`), whereas a numeric enum type also structurally accepts
any matching raw number. If you're already on a codebase/team that uses
`enum` heavily, staying consistent is reasonable — just know the quirks
above. `const enum` avoids the runtime object entirely (inlines values at
compile time) but the TypeScript team recommends avoiding it in library code
because of how unpredictably it behaves for consumers.

## Namespaces: legacy, mostly avoid

Namespaces predate ES modules and let you nest names to avoid global
collisions (`RecordStoreUtils.Album`, `RecordStoreUtils.Sales.recordSale`).
With ES modules now universal, namespaces are rarely the right choice for
new code — prefer regular module imports/exports. They still show up in
older codebases and in ambient `.d.ts` type declarations for global libraries
(see `typescript-strict-configuration-and-tooling` for `declare` and
declaration files).

## Common pitfalls

- **Forgetting `override`** and having a subclass method silently stop
  overriding its parent after a rename — turn on `noImplicitOverride` in
  `tsconfig.json` to make this a compile error everywhere, not just where you
  remember to type `override`.
- **Relying on `private`/`protected` for real security.** They're erased at
  compile time; the values are fully readable on the compiled object at
  runtime. Use `#field` for actual runtime privacy.
- **Choosing numeric enums for anything serialized or logged** — the
  reverse-mapping object and the numeric values are a poor fit for APIs,
  databases, or logs, where a string enum or `as const` string union reads
  far better and stays stable across enum reordering.
- **Assuming two enums (or an enum and its raw value) are interchangeable.**
  String enums are nominal; passing a matching plain string or a different
  enum with identical members is a type error.
- **Defaulting to `class` + inheritance for simple data.** If you don't need
  shared behavior or a contract enforced across unrelated types, a plain
  object type or discriminated union is often simpler than a class hierarchy.

## Checklist

- [ ] Every method that overrides a base class method is marked `override`,
      and `noImplicitOverride` is on in `tsconfig.json`
- [ ] `implements` is used to enforce a shape across unrelated classes;
      `extends`/`abstract` is used when shared implementation is also needed
- [ ] New code prefers `as const` objects or union types over `enum` unless
      the team has an existing enum convention
- [ ] If using `enum`, numeric vs. string enum was a deliberate choice (not
      the default), aware of the reverse-mapping and nominal-typing
      differences between them
- [ ] `private`/`protected` are understood as compile-time-only; genuine
      runtime privacy uses `#field` syntax
- [ ] No new namespace-based code organization — ES module imports/exports
      are used instead, except in ambient `.d.ts` files describing legacy
      globals
