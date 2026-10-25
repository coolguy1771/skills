---
name: typescript-narrowing-and-discriminated-unions
description: Model values that can be one of several shapes using TypeScript union types, literal types, and control-flow narrowing (typeof, instanceof, in, truthiness, discriminated unions, exhaustiveness checks with never). Use when the user asks how to type a value that "can be A or B", is designing a state machine or API response type ("loading/success/error"), has TypeScript errors like "Property X does not exist on type A | B" or "Object is possibly null/undefined", is stuck with optional properties that create "impossible states" (a bag of optionals), wants to know the difference between `unknown`, `any`, and `never`, or needs a switch statement that TypeScript verifies handles every case. Also load this when reviewing TS code with lots of optional (`?`) properties on one object instead of separate variants.
---

# TypeScript Narrowing and Discriminated Unions

TypeScript lets a value legitimately be "one of several types" via **union
types**, then progressively narrows that union down to a single member based
on the runtime checks you write. Most confusing "property doesn't exist"
errors come from not narrowing before use; most "impossible state" bugs come
from modeling a union as a single object with optional fields instead of an
actual union. This skill covers both.

## Union types and literal types

A union (`|`) means "this value is one of these types." Literal types narrow
that further to specific values, not just primitives:

```ts
type Id = string | number;

type HttpStatus = 200 | 404 | 500;          // literal number union
type AlbumGenre = "rock" | "country" | "electronic"; // literal string union

function logId(id: Id) { console.log(id); }
logId("abc"); logId(123); logId(true); // Error: boolean not assignable
```

Widen a union too far and you lose the specificity: `let genre = "rock"` on
its own infers as `string`, not `"rock"`, because `let` means the value could
later be reassigned to any string. `const genre = "rock"` infers the literal
type `"rock"` because it can never change. This "wide vs. narrow" distinction
governs whether a value is assignable to a literal-typed parameter — see
`typescript-object-types-and-immutability` for the full inference rules
around `let`/`const` and object properties.

## The wide/narrow hierarchy: `unknown`, `any`, `never`

```ts
const fn = (input: unknown) => {};
fn("hello"); fn(42); fn(null); fn([]);   // anything is assignable to unknown

const handle = (input: unknown) => {
  input.toUpperCase(); // Error: 'input' is of type 'unknown'
};
```

`unknown` is the widest type — anything can be assigned to it, but you can't
do anything with it until you narrow it. This is the correct type for "data
from the outside world" (API responses, JSON.parse, catch blocks) — **prefer
`unknown` over `any` for untyped input**. `any` isn't really a type at all:
it disables type checking entirely, in both directions. Think of it as
`unknown` meaning "I don't know what this is" and `any` meaning "I don't care
what this is."

`never` sits at the opposite end — the narrowest type, representing values
that can't occur. A function that always throws returns `never`, not `void`:

```ts
const fail = (): never => { throw new Error("always throws"); };

const accept = (input: never) => {};
accept(fail());        // OK — assigning never to never
const s: string = fail(); // OK — never is assignable to *everything*
```

You'll rarely write `never` yourself, but it's the backbone of exhaustiveness
checking (below) and shows up in narrowing error messages.

## Narrowing: turning wide types into narrow ones

Each of these techniques narrows a union based on a runtime check, and each
works only within the scope where the check is true:

```ts
// typeof — for primitives
function format(value: string | number) {
  if (typeof value === "string") return value.toUpperCase(); // string here
  return value.toFixed(2);                                    // number here
}

// truthiness — for null/undefined
function greet(name: string | null) {
  if (name) console.log(`Hello, ${name}`); // string here
}

// instanceof — for classes, including Error subclasses
try {
  somethingDangerous();
} catch (error) {
  // error is `unknown` in a catch clause (with useUnknownInCatchVariables)
  if (error instanceof Error) {
    console.error(error.message);
  } else {
    throw error; // re-throw non-Error throws instead of silently swallowing
  }
}

// `in` — for checking a property exists on an object union
type Circle = { radius: number };
type Square = { sideLength: number };
function area(shape: Circle | Square) {
  if ("radius" in shape) return Math.PI * shape.radius ** 2; // Circle here
  return shape.sideLength ** 2;                               // Square here
}

// throwing/returning early to narrow the rest of the function
function process(value: string | undefined) {
  if (value === undefined) throw new Error("value required");
  return value.toUpperCase(); // string here, for the rest of the function
}
```

Narrowing a genuinely `unknown` value with no shape info at all sometimes
requires a large manual conditional (`typeof value === "object" && value !==
null && "data" in value && ...`). This is legitimate but verbose; for
anything beyond a couple of checks, use a validation library (Zod, Valibot)
instead of hand-rolled narrowing — it gives you the same narrowed type from
one declarative schema.

## Discriminated unions: the fix for "bag of optionals"

The most common union-design mistake is modeling variants as **one type with
several optional properties** instead of **several types unioned together**:

```ts
// Before: a "bag of optionals" — allows impossible combinations
type State = {
  status: "loading" | "success" | "error";
  data?: string;
  error?: string;
};

const renderUI = (state: State) => {
  if (state.status === "error") {
    return `Error: ${state.error.toUpperCase()}`;
    //                     ^^^^^ Error: 'state.error' is possibly 'undefined'
  }
};

// Nothing stops this — an impossible state that type-checks fine:
const bad: State = { status: "loading", error: "oops", data: "also this" };
```

The `error` and `data` fields aren't actually tied to the `status` that
produces them, so TypeScript can't narrow one from the other, and the type
permits states that should never exist.

The fix: give each variant its own object type, sharing one literal
property — the **discriminant** — and union them:

```ts
// After: a discriminated union — impossible states are unrepresentable
type State =
  | { status: "loading" }
  | { status: "error"; error: string }
  | { status: "success"; data: string };

const renderUI = (state: State) => {
  if (state.status === "loading") return "Loading...";
  if (state.status === "error") return `Error: ${state.error.toUpperCase()}`; // string, narrowed
  return `Data: ${state.data}`; // string, narrowed
};
```

Checking `state.status === "error"` now narrows the *entire* union member,
not just one property — TypeScript knows that inside that branch, `state` is
specifically the `{ status: "error"; error: string }` variant. This works
because a discriminant must be a **literal type unique to each member**, not
a wider type like `string`.

`switch` statements are a natural fit for discriminated unions and pair with
exhaustiveness checking (below):

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; sideLength: number };

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.sideLength ** 2;
  }
}
```

Discrimination isn't limited to objects — a **tuple** can be discriminated by
its first element too, and destructuring preserves the narrowing relationship
between elements (this does *not* work for the equivalent object shape,
because destructured object properties lose their connection to each other):

```ts
type ApiResponse = ["error", string] | ["success", User[]];

async function handle(response: ApiResponse) {
  const [status, value] = response;
  if (status === "success") {
    value; // User[], narrowed via the tuple's first element
  }
}
```

## Exhaustiveness checking with `never`

Discriminated unions pay off further when you add a new variant later:
without exhaustiveness checking, a `switch` that forgets the new case simply
returns `undefined` silently. Force a compile error instead by assigning the
unhandled value to `never` in a `default` branch:

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; sideLength: number }
  | { kind: "triangle"; base: number; height: number }; // added later

function calculateArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.sideLength ** 2;
    default:
      // If every case above is handled, shape here is `never`.
      // Adding "triangle" without a case for it makes this a compile error:
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape kind: ${(_exhaustive as Shape).kind}`);
  }
}
```

This turns "I forgot to handle the new variant" from a silent runtime bug
into a build failure, at the exact call site that needs updating.

## Common pitfalls

- **Bag of optionals.** Several `?` properties on one type whose validity
  depends on another property's value — model it as a discriminated union
  instead so invalid combinations can't be constructed at all.
- **Reaching for `any` to silence an `unknown` error.** `any` disables
  checking permanently; narrow `unknown` with a real check (or a schema
  validator) so the rest of the function stays type-safe.
- **A discriminant that isn't a unique literal.** If two union members share
  the same literal value for the discriminant field, or the field is typed as
  `string` instead of a literal, narrowing silently stops working.
- **Expecting object destructuring to preserve narrowing the way tuple
  destructuring does.** `const { status, error } = state` does not let
  checking `status` narrow the type of the separately-destructured `error` —
  only accessing `state.error` after checking `state.status` narrows.
- **Adding a new union variant and not getting an error anywhere.** Add a
  `never`-typed exhaustiveness check in every `switch`/`if` chain over that
  union so new variants force you to update every call site.

## Checklist

- [ ] Values that are "one of several shapes" are modeled as unions of
      distinct types with a shared literal discriminant, not one type with
      several optional fields
- [ ] Untyped external input (`fetch` responses, `JSON.parse`, `catch`
      variables) is typed `unknown`, not `any`, and narrowed before use
- [ ] Every `switch`/`if` chain over a discriminated union has a `never`
      exhaustiveness check in its default/else branch
- [ ] Narrowing checks (`typeof`, `instanceof`, `in`, truthiness) are used
      instead of type assertions (`as`) wherever a real runtime check is
      possible
- [ ] `let` variables that must stay narrow are given an explicit literal
      union annotation, since plain inference widens them to `string`/`number`
