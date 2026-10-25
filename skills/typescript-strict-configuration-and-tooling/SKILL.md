---
name: typescript-strict-configuration-and-tooling
description: Set up and configure tsconfig.json correctly — the recommended base config, what each strict flag (strict, noUncheckedIndexedAccess, and friends) actually catches, choosing module/moduleResolution (NodeNext vs Preserve) based on whether tsc or a bundler transpiles, using TypeScript as a type-checking-only linter (noEmit) in CI, and typing untyped JS/globals with declaration files and `declare`. Use when the user is setting up a new TypeScript project or tsconfig.json, asks "what strict flags should I turn on", is confused why their bundler (Vite/esbuild/swc) doesn't catch type errors, sees "Cannot find module" for an untyped JS package, needs to add a global variable or type to `window`/`process.env`, or is deciding between `module: NodeNext` and `module: Preserve`. Also load this when reviewing a tsconfig.json for a new project or auditing one that's missing recommended strictness options.
---

# TypeScript Strict Configuration and Tooling

Most "why didn't TypeScript catch this" complaints trace back to tsconfig
settings, not the type system itself — a missing strict flag, a bundler that
transpiles without type-checking, or an untyped global with no declaration.
This skill covers the recommended base configuration, what strictness
options actually do, how TypeScript fits into a build pipeline that includes
a bundler, and how to type things that aren't TypeScript at all (globals,
untyped JS modules).

## The problem `tsc` solves, and the one it doesn't

Browsers and Node can't run TypeScript directly — type annotations aren't
valid JavaScript syntax. `tsc` transpiles TypeScript to JavaScript by
stripping types, and (separately) type-checks your code. These are two
different jobs, and modern tooling often splits them between two different
tools:

```ts
const run = (message: string) => console.log(message);
// browsers error on `: string` directly — it must be stripped before running
```

Modern bundlers (Vite, esbuild, swc) and runtimes (Bun, newer Node) can
**transpile** TypeScript (strip the types) but do **not** type-check it —
they'll happily run code with type errors, including straight into
production. `tsc` is still required for the actual type checking, even in a
project that never runs `tsc` to produce its shipped JavaScript.

## Recommended base `tsconfig.json`

```json
{
  "compilerOptions": {
    /* Base */
    "skipLibCheck": true,
    "target": "es2022",
    "esModuleInterop": true,
    "allowJs": true,
    "resolveJsonModule": true,
    "moduleDetection": "force",
    "isolatedModules": true,

    /* Strictness */
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

What each does:

- **`skipLibCheck`** — skips type-checking `.d.ts` files (including
  `node_modules`), which speeds up compilation and avoids errors in
  third-party type definitions you don't control.
- **`target`** — the ECMAScript version emitted. Only affects **syntax**
  transformation (optional chaining, etc. get downleveled); it does **not**
  polyfill missing runtime APIs (`.replaceAll`, etc.) for older targets —
  that's a separate concern (`lib`, and actual polyfills).
- **`esModuleInterop`** — aligns TypeScript's CommonJS/ESM interop behavior
  with Babel/webpack's; turn this on for essentially every application.
- **`allowJs`** — lets `.js` files be imported into a TS project without
  their own declarations, useful for gradual adoption.
- **`resolveJsonModule`** — allows `import data from "./data.json"`.
- **`moduleDetection: "force"`** — treats every `.ts` file as a module
  (local scope) even if it has no `import`/`export`. Without this,
  TypeScript guesses module-vs-script per file by checking for
  import/export statements, which causes surprising global-scope collisions
  in files that happen to have neither (see "Modules vs. scripts" below).
- **`isolatedModules`** — ensures every file could be transpiled
  independently, with no cross-file type information required. Required for
  bundlers (esbuild, swc, Babel) that transpile file-by-file; also catches a
  few TypeScript features (like `const enum` re-exports) that don't survive
  single-file transpilation.
- **`strict`** — turns on the standard bundle of strictness checks
  (`noImplicitAny`, `strictNullChecks`, and others). Turn this on for every
  new project; see "Gradual adoption" below for existing codebases.
- **`noUncheckedIndexedAccess`** — makes every array/index access return
  `T | undefined` instead of `T`, catching a real class of runtime crash:

  ```ts
  const tracks: string[] = ["a", "b"];
  const track = tracks[5];             // without the flag: typed `string` (wrong!)
  track.toUpperCase();                 // crashes at runtime — index 5 doesn't exist
  // with noUncheckedIndexedAccess: track is `string | undefined`,
  // and TypeScript forces you to handle the missing case before calling `.toUpperCase()`
  ```

  This flag is not part of `strict` itself and is worth turning on
  separately — it catches real bugs at the cost of needing `undefined`
  checks around array/record indexing.

## Additional strictness options beyond the base

Once `strict` + `noUncheckedIndexedAccess` are on, consider these
individually:

- **`noImplicitOverride`** — requires the `override` keyword on methods that
  override a base class method (see `typescript-classes-and-enums`).
- **`noUnusedLocals`** / **`noUnusedParameters`** — flag declared-but-unused
  variables/parameters.
- **`noImplicitReturns`** — every code path in a function must return a
  value if any path does.
- **`noFallthroughCasesInSwitch`** — every non-empty `case` must end in
  `break`/`return`/`throw`.
- **`exactOptionalPropertyTypes`** — an optional property (`x?: string`)
  must be either present with a real value or absent entirely; assigning
  `undefined` explicitly becomes an error, distinguishing "not set" from
  "set to undefined."
- **`noPropertyAccessFromIndexSignature`** — forces `obj['dynamicKey']`
  instead of `obj.dynamicKey` for properties that come from an index
  signature, keeping "known property" access visually distinct from
  "dynamic property" access.

## Deciding `module`: `NodeNext` vs. `Preserve`

This is the single most consequential decision beyond the base config, and
it's a straight either/or based on **who does the actual transpiling**:

- **`tsc` itself transpiles your code** (you run `tsc` to produce the shipped
  JS) → set `"module": "NodeNext"`. This emulates Node's real module
  resolution (including `package.json` `exports`), and requires writing
  `.js` extensions on relative imports even in `.ts` files
  (`import { Album } from "./album.js"`) — because TypeScript doesn't
  rewrite import paths, only strips types, so the source must already look
  like the emitted JS.
- **A bundler transpiles your code** (Vite, webpack, esbuild, Rollup,
  Next.js, SvelteKit, etc.) → set `"module": "Preserve"` and
  `"noEmit": true`. The bundler owns module resolution; `tsc` in this setup
  is a **type-checking-only linter**, never asked to emit JavaScript. No
  `.js` extensions needed on relative imports — the bundler resolves them.

Mixing these up produces confusing errors: `NodeNext` extension requirements
show up as spurious errors under a bundler setup, and `Preserve` without
`noEmit` produces a `dist/` folder nobody asked for and nobody reads.

## `tsc` as a CI gate (type-check without emitting)

Whether or not `tsc` produces your shipped JavaScript, run it in CI as a
type-check-only step:

```json
{ "compilerOptions": { "noEmit": true } }
```

```jsonc
// package.json
{ "scripts": { "typecheck": "tsc --noEmit" } }
```

This catches type errors a bundler would silently ship, and is the standard
way to gate a PR merge on "no type errors" independent of whatever tool
actually builds the app. `tsc --watch` is the equivalent for local dev
feedback without re-running manually on every save.

## Library-specific options

If you're publishing a package (not just building an application), add:

- **`declaration: true`** — emits `.d.ts` files alongside the compiled JS so
  consumers get types.
- **`declarationMap: true`** and **`composite: true`** — needed together in
  a monorepo, so consumers can "go to definition" through to your original
  `.ts` source instead of the emitted `.d.ts`.
- **`outDir`** — where compiled output goes (keep it out of version control).

## Modules vs. scripts, and why an empty-looking file errors

A `.ts` file with no `import`/`export` is treated as a **script** (global
scope) by default, not a module — this is a legacy default from before ES
modules existed. This causes real, confusing bugs:

```ts
// utils.ts, with no imports/exports anywhere in the file
const name = "Alice"; // Error: Cannot redeclare block-scoped variable 'name'
```

This collides with the DOM's global `window.name`, because TypeScript
treated the file as a script, putting `name` in the global scope. Fix it by
forcing module treatment — either add `export {}` to the specific file, or
set `"moduleDetection": "force"` project-wide (included in the recommended
base config above) so every `.ts` file is always treated as a module
regardless of whether it happens to have imports/exports yet.

## Declaration files and `declare`: typing things that aren't TypeScript

`.d.ts` files describe the shape of JavaScript (or ambient globals) without
containing any implementation — `declare` marks a value as existing without
providing its definition:

```ts
// global.d.ts — a script (no imports/exports), so this is added to the global scope
declare const ALBUM_API: {
  getAlbumInfo(upc: string): Promise<{ title: string }>;
};

// Typing an untyped third-party module you don't control:
declare module "duration-utils" {
  export function formatDuration(seconds: number): string;
}
```

`declare global { ... }` lets you add to the global scope **from inside a
module** (a file that does have imports/exports) — useful for colocating a
global type declaration with the code that actually uses it, instead of a
separate ambient file. Extending existing global types (like adding a custom
property to `process.env` or `window`) uses the same `declare global`
pattern with an interface merge:

```ts
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string; // now process.env.DATABASE_URL is typed as string
    }
  }
}
export {}; // needed to make this file a module, so `declare global` is legal here
```

For untyped packages from npm, check DefinitelyTyped
(`@types/<package>`) before hand-authoring a `.d.ts` — most popular
untyped packages already have community-maintained types available as a
separate install.

## Gradual strictness adoption for an existing codebase

Turning on `strict` in one PR on a large, previously-loose codebase usually
produces hundreds of errors at once. Prefer incremental adoption:

1. Turn on `strict` in `tsconfig.json`, but scope it down with `include`/a
   separate `tsconfig` that only covers new/actively-worked-on directories,
   or use per-file `// @ts-nocheck` as a *temporary, tracked* escape hatch
   on the noisiest legacy files (never as a permanent fix — see
   `typescript-assertions-and-type-system-gotchas` for why `@ts-nocheck` is
   otherwise best avoided).
2. Fix errors file-by-file, removing the escape hatch as each file clears.
3. Add `noUncheckedIndexedAccess` and the other individual strictness flags
   only after `strict` itself is fully adopted — they add real,
   often-substantial additional work.

Don't leave a codebase half-migrated indefinitely — an escape hatch that
never shrinks defeats the purpose of turning `strict` on at all.

## Common pitfalls

- **Relying on a bundler's dev server to catch type errors.** It won't —
  Vite/esbuild/swc transpile without type-checking; run `tsc --noEmit`
  separately, ideally in CI.
- **Setting `module: NodeNext` under a bundler setup** (or vice versa),
  producing spurious import-extension errors or an unwanted `dist/` output.
- **Skipping `moduleDetection: "force"`** and hitting mysterious global-scope
  collisions in a file that happens to have no imports/exports yet.
- **Turning on `strict` for an entire large legacy codebase in one shot**
  with no incremental plan, stalling the migration indefinitely.
- **Hand-authoring `.d.ts` files for popular npm packages** without first
  checking whether `@types/<package>` already exists on DefinitelyTyped.
- **Forgetting `noUncheckedIndexedAccess`** and trusting array/record index
  access to always return a defined value — it's opt-in, not part of
  `strict`.

## Checklist

- [ ] `tsconfig.json` includes the full recommended base
      (`skipLibCheck`, `target`, `esModuleInterop`, `allowJs`,
      `resolveJsonModule`, `moduleDetection: "force"`, `isolatedModules`,
      `strict`, `noUncheckedIndexedAccess`)
- [ ] `module` is `NodeNext` (tsc transpiles) or `Preserve` + `noEmit`
      (bundler transpiles) — deliberately, not left at the `tsc --init`
      default
- [ ] CI runs `tsc --noEmit` (or equivalent) as a required check, independent
      of whatever tool actually builds/bundles the app
- [ ] Untyped third-party packages are checked against `@types/<package>`
      before writing a hand-authored `.d.ts`
- [ ] Global augmentations (`process.env`, `window`) use `declare global`
      with an interface merge, not scattered `as any` casts at each use site
- [ ] An existing codebase adopting `strict` has an incremental plan
      (per-directory or per-file), not a single unscoped flip with a
      permanent escape hatch
