---
title: "strip-freeform-comments-ar-connection-adapters-toplevel-tests"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
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

Follow-up slice of `strip-freeform-comments-ar-connection-adapters-toplevel`.
That story swept the **implementation** files at the root of
`packages/activerecord/src/connection-adapters/` (postgresql-adapter.ts 77,
sqlite3-adapter.ts 46, mysql2-adapter.ts 41, abstract-adapter.ts 34,
abstract-mysql-adapter.ts 13, schema-cache.ts 10, adapter-args.ts 5, and the
remaining small src files) — ~625 LOC, at the ceiling — and registered
`packages/activerecord/src/connection-adapters/*.ts` in `eslint.config.mjs`'s
`no-freeform-comments` block with
`ignores: ["packages/activerecord/src/connection-adapters/*.test.ts"]`.

Remaining: the **top-level test files** in that same directory, ~145 rule
findings. Heaviest: `schema-cache.test.ts` (27),
`postgresql-adapter.exec-query.trails.test.ts` (14),
`postgresql-adapter.get-client.trails.test.ts` (12),
`sqlite3-introspection.trails.test.ts` (11),
`abstract-mysql-adapter.trails.test.ts` (10),
`postgresql-adapter.type-map.trails.test.ts` (7), then a long tail of 1–5.

Measure with `pnpm exec eslint
'packages/activerecord/src/connection-adapters/*.test.ts' --rule
'{"blazetrails/no-freeform-comments":["warn",{"report":true}]}'`. Autofix with
the same rule WITHOUT `{report:true}` — the `report` option suppresses the fix.

When done, delete the `ignores` line from that block so the rule covers the
whole top-level directory.

The bar: a comment that restates the line or branch it sits on goes. What
survives, survives as JSDoc carrying a tag or a Rails citation. Rails' OWN
comments are deleted too. Deferred work becomes a story.

## Acceptance criteria

- [ ] `pnpm eslint --fix` applied to the top-level `*.test.ts` files and the
      deletions reviewed.
- [ ] The `ignores` entry removed from the connection-adapters top-level
      `no-freeform-comments` block in `eslint.config.mjs`.
- [ ] `pnpm eslint` clean over the whole top-level directory; a second `--fix`
      is a no-op.
- [ ] `pnpm typecheck` clean; the test files touched run green.
- [ ] Deferred work found in a deleted comment is filed as its own story.
