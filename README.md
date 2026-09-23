# Contract Validation Smoke Test

**What it teaches:** how to validate the contract package independently,
before any backend implementation starts. If the shared contract
(`packages/contract`) is wrong, everything built on top of it — backend
routes, frontend types — inherits the bug. Testing it in isolation, with
no dependency on the backend or frontend being built yet, catches that
early.

This README documents, step by step, what was done to satisfy the
prompt below and how the result was verified.

> Add a small Vitest suite in `packages/contract` that checks:
> - `STATES` includes all six expected values.
> - `ACTIONS` includes all five actions.
> - `PRIORITIES` includes all four priorities.
> - no array contains duplicates.
> - the package exports the generated type module without runtime errors.
>
> Run only the contract tests and show the result.

## Step 1 — Read the package before touching it

Before writing a test, we looked at what `packages/contract` already
contains:

- **`src/index.ts`** — hand-written. It re-exports types generated from
  the OpenAPI spec and defines three runtime constant arrays that must
  stay in sync with those types: `STATES`, `ACTIONS`, `PRIORITIES`. A
  compile-time-only check (`AssertSame<...>`) already guards that the
  *type-level* union and the array contents match — but that check runs
  only when the TypeScript compiler runs, and it doesn't catch things
  like accidental duplicate entries or count mismatches. That's the gap
  a runtime test suite fills.
- **`src/types.gen.ts`** — generated from `openapi.yaml` via
  `npm run gen` (which shells out to `openapi-typescript`). It contains
  only type declarations, no runtime code.
- **`package.json`** — had a `gen` script but no `test` script, and no
  test runner installed.
- **The workspace root** — no Vitest config existed anywhere in the
  monorepo yet. This was the first package to introduce it.

## Step 2 — Turn each requirement into a concrete assertion

| Prompt requirement | How it was tested |
| --- | --- |
| `STATES` includes all six expected values | `expect(STATES).toEqual(expect.arrayContaining([...6 values]))` + `toHaveLength(6)` |
| `ACTIONS` includes all five actions | same pattern, 5 values |
| `PRIORITIES` includes all four priorities | same pattern, 4 values |
| No array contains duplicates | a small `hasNoDuplicates()` helper (`new Set(values).size === values.length`), applied to each of the three arrays |
| Package exports the generated type module without runtime errors | `await expect(import("./types.gen.js")).resolves.toBeDefined()` |

Using `arrayContaining` together with `toHaveLength` checks two things at
once: every expected member is present, *and* there are exactly that many
members — so the test fails if a value goes missing **or** if a stray
extra value sneaks in, without being sensitive to array ordering.

## Step 3 — Install Vitest for this package

```bash
npm install --workspace @equipment-hub/contract vitest --save-dev
```

This adds `vitest` to `packages/contract/package.json`'s
`devDependencies` and to the root `package-lock.json`. A `test` script
was added alongside the existing `gen` script:

```json
"scripts": {
  "gen": "openapi-typescript openapi.yaml -o src/types.gen.ts",
  "test": "vitest run"
}
```

`vitest run` (rather than plain `vitest`, which watches by default) is
what you want for a single, CI-style pass.

## Step 4 — Write the suite

The suite lives at `src/index.test.ts`, next to the code it exercises,
and imports directly from `./index.js` — Vitest's resolver maps that back
to `index.ts` at runtime, so no build step is needed first.

Layout:

- One `describe` block per export (`STATES`, `ACTIONS`, `PRIORITIES`),
  each with an "includes all expected values" test and a "contains no
  duplicates" test.
- One `describe` block for the generated type module, with a single test
  that dynamically imports `types.gen.ts` and asserts the import
  resolves (rather than rejects/throws).

## Step 5 — Run only the contract package's tests

From the monorepo root, scoped with `--workspace` so it doesn't try to
run tests in every package:

```bash
npm run test --workspace @equipment-hub/contract
```

**Result:**

```
> @equipment-hub/contract@0.0.0 test
> vitest run

 ✓ src/index.test.ts (7 tests) 11ms

 Test Files  1 passed (1)
      Tests  7 passed (7)
```

All 7 assertions pass:

1. `STATES` includes all six values
2. `STATES` has no duplicates
3. `ACTIONS` includes all five values
4. `ACTIONS` has no duplicates
5. `PRIORITIES` includes all four values
6. `PRIORITIES` has no duplicates
7. `types.gen.ts` imports without a runtime error

## Why this approach

- **No global Vitest config was added.** Since this was the first
  Vitest usage in the repo, the change was kept local to
  `packages/contract` instead of guessing how `apps/backend` and
  `apps/frontend` should eventually be configured.
- **`scripts/verify-contract.mjs` was left untouched.** That script
  solves a different problem — detecting drift between `openapi.yaml`
  and the committed `types.gen.ts` — and already runs separately via
  `npm run verify:contract`. It isn't a test, so it wasn't folded into
  the Vitest suite.
- **Assertions target the contract's values, not the implementation.**
  The tests check *what* `STATES`, `ACTIONS`, and `PRIORITIES` must
  contain, not *how* `index.ts` builds them — so refactoring the
  implementation won't break the suite, but a real change to the
  contract will.
