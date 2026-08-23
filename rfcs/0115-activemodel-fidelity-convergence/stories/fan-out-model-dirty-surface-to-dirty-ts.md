---
title: "Fan out the ActiveModel dirty surface from model.ts to dirty.ts"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-validates-of-macros-to-helper-methods
deps-rfc: []
est-loc: 300
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

After the ActiveRecord save-side family is removed
(`move-ar-save-side-dirty-surface-out-of-model`), twenty `Model` members
(78 code lines) remain whose Rails home is
`vendor/rails/activemodel/lib/active_model/dirty.rb`:

`changed` `model.ts:2158` (`dirty.rb:286`), `changedAttributes` `:2176`
(`:343`), `changes` `:2180` (`:353`), `attributeChanged` `:2184` (`:300`),
`attributeWas` `:2205` (`:305`), `attributeChange` `:2210`,
`previousChanges` `:2223` (`:363`), `attributePreviouslyChanged` `:2323`
(`:310`), `attributePreviouslyWas` `:2333` (`:315`), `restoreAttributes`
`:2337` (`:320`), `attributeWillChange` `:2352`, `restoreAttribute` `:2362`,
`attributePreviousChange` `:2378`, `changesApplied` `:2382` (`:272`),
`clearChangesInformation` `:2392` (`:325`), `clearAttributeChanges` `:2401`
(`:331`), `mutationsFromDatabase` `:2412`, `mutationsBeforeLastSave` `:2423`,
`forgetAttributeAssignments` `:2435`, `clearAttributeChange` `:2447`,
`attributeChangedInPlace` `:2712` (`:367`).

`packages/activemodel/src/dirty.ts` already defines every one of these names
(`:184`, `:189`, `:194`, `:237`, `:250`, `:263`, `:286`, `:334`, `:351`,
`:368`, `:373`, `:603`, `:646`, `:660`, …). The `Model` copies are wrappers
around `this._dirty`, sometimes with extra behaviour bolted on — e.g.
`model.ts:2184` `attributeChanged` is 10 lines that re-implement the
`from:`/`to:` option matching `dirty.ts` and
`attribute_mutation_tracker.rb:44` already do.

So this is a **shadow deletion**, not a move: `Model` should reach `dirty.ts`
through `include()` / `Included<>` and define nothing.

Two names to watch: `attributeWillChange` and `restoreAttribute` are `novel` in
`pnpm parity:api:extra --package activemodel` — Rails' spellings are
`attribute_will_change!` and `restore_attribute!` (both bang), and `dirty.ts`
has `attributeWillChangeBang` (`:92`) and `restoreAttributeBang` (`:646`)
already. Converge the non-bang `Model` spellings onto the bang names rather
than carrying them across.

Depends on the ActiveRecord save-side story landing first, otherwise the two
PRs collide on the same region of `model.ts`.

## Acceptance criteria

- `model.ts` defines no dirty member; `Model` gets them from `dirty.ts` via
  `include()` / `Included<>`.
- `attributeWillChange` and `restoreAttribute` are gone in favour of the
  existing `*Bang` names; call sites updated.
- Where a `Model` wrapper carried behaviour the `dirty.ts` body lacks (the
  `from:`/`to:` matching, alias resolution via `resolveAliasName`), that
  behaviour lands in the Rails-named body it belongs to, not in a new wrapper.
- `pnpm parity:api:extra --package activemodel` loses ~20 rows from `model.ts`,
  including both `novel` ones.
- Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean, no
  reseed; stranded `activemodel/dirty.json` rows hand-deleted then tightened.

## Verification

```bash
pnpm vitest run packages/activemodel/src/dirty.test.ts packages/activemodel/src/dirty.trails.test.ts packages/activemodel/src/dirty-mutations.test.ts packages/activemodel/src/dirty-generated-methods.test.ts packages/activemodel/src/attributes-dirty.test.ts
```
