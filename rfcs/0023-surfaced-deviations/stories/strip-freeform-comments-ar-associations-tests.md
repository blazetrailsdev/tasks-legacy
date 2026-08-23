---
title: "strip-freeform-comments-ar-associations-tests"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
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

`strip-freeform-comments-ar-associations` swept the implementation files under
`packages/activerecord/src/associations/**` and registered the
`blazetrails/no-freeform-comments` rule for that tree in `eslint.config.mjs`,
with `ignores: ["packages/activerecord/src/associations/**/*.test.ts"]` — the
test files under the same tree are the remaining backlog (~500 flagged blocks,
too large for one PR alongside the sources).

Measured leaders: `has-many-associations.test.ts` (110),
`collection-proxy.trails.test.ts` (49), `eager.test.ts` (28),
`association-scope.trails.test.ts` (28),
`has-many-through-associations.test.ts` (25).

The bar is the one from slice 1: a comment that restates the line or branch it
sits on goes, whatever its subject. What survives, survives as JSDoc carrying a
tag or a Rails citation. Rails' own comments go too (the Ruby is vendored).
Test NAMES must not change.

Split into several PRs of its own; each one narrows the `ignores` list rather
than dropping it wholesale.

## Acceptance criteria

- [ ] The `ignores` entry for
      `packages/activerecord/src/associations/**/*.test.ts` is gone from
      `eslint.config.mjs`.
- [ ] `pnpm eslint` clean over the tree, and a second `--fix` run is a no-op.
- [ ] The swept test files run green; no test name changed.
- [ ] Any deferred work found in a deleted comment is filed as its own story.
