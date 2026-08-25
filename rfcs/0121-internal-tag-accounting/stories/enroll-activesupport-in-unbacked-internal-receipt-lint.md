---
title: "enroll-activesupport-in-unbacked-internal-receipt-lint"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

RFC 0121 landed the reverse lint in
`eslint/unbacked-internal-needs-receipt.mjs`: a public TS declaration carrying
`@internal` whose (file, name) is absent from `eslint/rails-private-methods.json`
must ALSO carry a `@noRailsEquivalent PERMANENT|CONVERGEABLE <reason>` receipt,
which wins in `extract-ts-api.ts` (`internalJsDocTagApplies`) so the member
re-enters the measured surface as `Allowed` instead of vanishing.

The rule ships behind a per-package enrollment set — its `files` list in
`eslint.config.mjs` and `eslint/rails-private-jsdoc.config.mjs`, which must stay
in sync. That set is **only-grow**. `trailties` was the first package enrolled;
this story enrolls `activesupport`.

The work: build the manifest (`pnpm parity:api` then
`pnpm rails-privates:manifest`), add `"packages/activesupport/src/**/*.ts"` to both
`files` lists, and resolve every flagged declaration. Three outcomes, in this
order of preference:

1. **The name has a public Rails counterpart** — the `@internal` was never
   backed. Delete the tag. (In trailties this covered `tsortEachChild`,
   `allAutoloadPaths` and friends.)
2. **The name is a trails misspelling of a private Rails one** — rename it to
   the manifest's spelling. (trailties: `allLoadPaths` -> `_allLoadPaths`, Rails
   `_all_load_paths`, engine.rb:730.)
3. **Genuinely extra surface** — add a reviewed one-line reason opening
   `PERMANENT` or `CONVERGEABLE`, citing the Rails `file:line` it stands in for.

Traps found while doing trailties, all of which apply here:

- A tag on a name that does not actually flag as extra is **STALE** and fails
  `pnpm parity:api:extra` for the whole repo, not just this package. Names that
  score as **moved** (present in Rails, just in another .rb) cannot be tagged
  either — they converge by moving the port, not by a receipt.
- `arel` and `activerecord` are gated by `pnpm parity:api:extra:gate`
  (`scripts/api-compare/extra-surface-mark.json`), which is only-shrink with no
  reseed. Un-`@internal`-ing a name ADDS it to novel/total; retagging does not
  (allowlisted names are subtracted first). Measure before committing, and note
  that a file whose exported functions were ALL `@internal` has no fabricated
  module entry — retagging one of them makes that module name appear as a new
  novel name.

## Acceptance criteria

- `"packages/activesupport/src/**/*.ts"` added to the `unbacked-internal-needs-receipt`
  block in BOTH `eslint.config.mjs` and `eslint/rails-private-jsdoc.config.mjs`.
- `pnpm exec eslint --no-inline-config -c eslint/rails-private-jsdoc.config.mjs "packages/activesupport/src/**/*.ts"`
  reports zero `unbacked-internal-needs-receipt` violations.
- Every added `@noRailsEquivalent` reason opens `PERMANENT` or `CONVERGEABLE`
  and cites a Rails `file:line`.
- `pnpm parity:api:extra` reports no STALE tag, `pnpm parity:api:extra:gate` is
  green, and `scripts/api-compare/extra-surface-mark.json` is unchanged.
- `pnpm parity:api` / `pnpm parity:test` deltas are non-negative.
