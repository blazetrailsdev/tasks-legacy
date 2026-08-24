---
title: "strip-freeform-comments-ar-connection-adapters-toplevel"
status: claimed
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T14:09:30Z"
assignee: "strip-freeform-comments-ar-connection-adapters-toplevel"
blocked-by: null
closed-reason: null
---

## Context

Slice 4 of `strip-freeform-comments-ar-connection-adapters`. Slice 2 swept the
`{mysql,mysql2,sqlite3,postgresql}/**` sub-globs; slice 3 sweeps `abstract/**`.

Remaining: the **top-level files** of
`packages/activerecord/src/connection-adapters/*.ts` — measured 382 rule
findings, the largest group, so it needs splitting again. Heaviest files:
`postgresql-adapter.ts` (77), `sqlite3-adapter.ts` (46), `mysql2-adapter.ts`
(41), `abstract-adapter.ts` (34), `schema-cache.test.ts` (27),
`postgresql-adapter.exec-query.test.ts` (14), `abstract-mysql-adapter.ts` (13),
`postgresql-adapter.get-client.test.ts` (12). A single PR over all of them was
measured at ~820 changed LOC, over the ceiling; the three concrete adapter
files alone are ~440.

Measure with `pnpm exec eslint '<glob>' --rule
'{"blazetrails/no-freeform-comments":["warn",{"report":true}]}'`. Register the
files a group at a time in `eslint.config.mjs`'s `no-freeform-comments` block
(the block's `files` list accepts individual paths as well as globs), so each
PR leaves the rule green.

The bar: a comment that restates the line or branch it sits on goes. What
survives, survives as JSDoc carrying a tag or a Rails citation. Rails' OWN
comments are deleted too. Deferred work becomes a story.

## Acceptance criteria

- [ ] Every `packages/activerecord/src/connection-adapters/*.ts` top-level file
      registered in the `no-freeform-comments` block (across as many PRs as the
      LOC ceiling needs, each one green).
- [ ] `pnpm eslint --fix` applied and the deletions reviewed.
- [ ] `pnpm eslint` clean over the registered paths; a second `--fix` is a no-op.
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Deferred work found in a deleted comment is filed as its own story.
