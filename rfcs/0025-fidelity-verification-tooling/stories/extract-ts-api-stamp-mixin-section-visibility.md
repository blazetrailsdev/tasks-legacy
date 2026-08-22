---
title: "extract-ts-api: stamp visibility from defineModule sections"
status: draft
updated: 2026-08-20
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: ["activerecord"]
deps:
  - converge-public-instance-methods-onto-one-helper
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/extract-ts-api.ts` derives a member's `visibility` from TS
syntax (`extract-ts-api.ts:791`, `:2064`), and sets
`internal = visibility !== "public" || hasInternalJsDocTag(...)`. Top-level
exported functions are unconditionally `visibility: "public"`
(`extract-ts-api.ts:712`, `:964`, `:1923`).

That is wrong for the mixin files. The ~40 functions listed in
`QueryMethodsPrivateInstanceMethods`
(`packages/activerecord/src/relation/query-methods.ts:2750-2790`), the five in
`QueryMethodsProtectedInstanceMethods` (`:2730-2740`), and `relationWith` in
`SpawnMethodsPrivateInstanceMethods`
(`packages/activerecord/src/relation/spawn-methods.ts:159-161`) are all
`private` or `protected` in Rails — `query_methods.rb:1604`, `:1663`, `:1677`,
`spawn_methods.rb:71` — but are ordinary exported top-level functions in TS
(they must be, so the mixin object can reference them). So they extract as
public, and `parity:api:extra` plus the coverage totals currently count
Rails-private helpers as public surface.

The existing mechanism for this is documentary, not structural:
`eslint/rails-private-jsdoc.mjs` reads the generated
`eslint/rails-private-methods.json` (built by
`scripts/build-rails-privates-manifest.ts`, which resolves effective Ruby
visibility through the include graph) and requires an `@internal` JSDoc tag. A
comment is not something the extractor reads for visibility.

The sibling story
`0082-ruby-ts-idiom-conversion-classes/converge-public-instance-methods-onto-one-helper`
introduces `defineModule(publicSection, protectedSection, privateSection)` in
`packages/activesupport/src/include.ts` as the single declaration site for
mixin-member visibility. That gives the extractor ONE syntactic shape to
recognize, rather than a naming convention spread across sibling exported
consts — which is the concrete tooling argument for that shape and the reason
this story depends on it.

## Acceptance criteria

- `extract-ts-api.ts` recognizes a `defineModule(...)` call and, for each
  top-level function referenced by its protected / private section object
  literals (including shorthand, renamed `key: fn`, and alias entries such as
  `buildHavingClause: buildWhereClause`), stamps `visibility: "protected"` /
  `"private"` and `internal: true` on that function's extracted entry.
- Section membership is resolved within the module's own file; a reference the
  extractor cannot resolve to a same-file top-level function is reported, not
  silently skipped.
- `EXTRACTOR_OUTPUT_FIELDS` in `scripts/api-compare/extractor-schema.ts` is
  updated if any emitted key changes, so the ts-api cache token changes and
  stale entries are evicted (see PR #4020 precedent noted at
  `scripts/parity/types.ts:56-58`).
- `pnpm parity:api:extra --package activerecord` totals are UNCHANGED, and the
  PR body says so. (Corrected 2026-08-22: an earlier draft of this criterion
  claimed the totals would move. They should not. The Ruby side of the allowed
  set already includes private and protected methods —
  `extra-surface.ts:1199-1207` — so a TS public port of a Rails-private method
  is already allowed and was never counted as extra. This story fixes
  `internal` ACCURACY, which the visibility gate and the private-surface story
  depend on; it is not an extra-surface reduction. A total that DOES move here
  is a bug worth investigating, not the goal.)
- `pnpm parity:api` / `pnpm parity:test` deltas are non-negative.
- Unit cover in `scripts/api-compare/extract-ts-api.test.ts` for all three
  section kinds plus the alias-entry case.

## Notes

Pure accuracy fix — no new gate here. The gate that consumes this is the
sibling story `add-visibility-parity-gate`.
