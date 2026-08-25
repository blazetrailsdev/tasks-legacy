---
title: "Relocate to-sql.test.ts's trails-only extras to to-sql.trails.test.ts"
status: ready
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/visitors/to-sql.test.ts` is the mirror of
`vendor/rails/activerecord/test/cases/arel/visitors/to_sql_test.rb`, but roughly
**93 of its `it(...)` cases have no counterpart there** — they are trails-only
extras that grew inside the Rails-mirror file instead of in the sibling
`to-sql.trails.test.ts` (the repo convention: TS-only extras live in the
`*.trails.test.ts` twin).

Measured in PR #7041 while classifying every `new Visitors.ToSql(...)` site: of
the 124 sites, 32 sat in tests whose names appear verbatim in `to_sql_test.rb`;
the rest are trails-only — `compileWithBinds extracts bind values`,
`GreaterThan short-circuits to 1=0 for positive unboundable`,
`renders DELETE FROM via the table visitor`, the `Quoted`/`BindParam` groups,
the `Rails-mirrored private helpers` and `Rails-mirrored to_sql tail helpers`
blocks, the whole `ArelQuoter / defaultQuoter wiring` describe, and so on.

That story's converged shape named the relocation explicitly ("Ideally the
trails-only ones relocate to `to-sql.trails.test.ts`, so the Rails-mirroring
file needs only the one shared visitor") but it did not fit under the PR's LOC
ceiling alongside the connection reclassification, so only the classification
shipped.

Keeping them in the mirror file has a concrete cost: `parity:test` counts them
as `extra (TS only)` against `to_sql_test.rb` (the arel package currently reports
427 such extras), and every future reader of the file has to re-derive which
half is Rails and which is ours — the same audit PR #7041 had to do from
scratch.

## Converged shape

- Every `it(...)` in `to-sql.test.ts` whose name is not in `to_sql_test.rb`
  moves verbatim to `packages/arel/src/visitors/to-sql.trails.test.ts` — no
  rename (test names are how `parity:test` matches), no assertion rewrite.
- The blocks that exist to exercise the ActiveRecord abstract-adapter quoter
  (`defaultQuoter` / `testConnection`) move as whole describes, carrying the
  call-site notes PR #7041 added.
- What remains in `to-sql.test.ts` mirrors `to_sql_test.rb` one-for-one and
  needs only the shared `visitor` on `fakeRecordConnection`
  (`helper.rb:31-33`, `to_sql_test.rb:10-14`).
- Split across PRs by describe-block if the move exceeds the LOC ceiling.

## Acceptance criteria

- [ ] `to-sql.test.ts` contains only tests whose names appear in
      `to_sql_test.rb`; the trails-only ones live in `to-sql.trails.test.ts`.
- [ ] No test name changes; `pnpm vitest run packages/arel` green.
- [ ] `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry does not
      regress, and the arel `extra (TS only)` count in `pnpm parity:test` drops
      by the number of relocated tests.
