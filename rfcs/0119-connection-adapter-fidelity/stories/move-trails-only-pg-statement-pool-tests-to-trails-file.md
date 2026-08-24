---
title: "Move trails-only cases out of the Rails-mapped PG statement-pool test file"
status: draft
updated: 2026-08-13
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Noticed while touching these suites in PR #6455.

`packages/activerecord/src/adapters/postgresql/statement-pool.test.ts` is a
Rails-mapped test file — `parity:test` matches it against
`vendor/rails/activerecord/test/cases/adapters/postgresql/statement_pool_test.rb`,
whose only test is `test_cache_is_per_pid`. Everything else in the trails file
is trails-only content, including:

- `"rejects non-boolean preparedStatements at construction time and via assignment"`
- `"reads statementLimit from the options hash"` / `"reads preparedStatements
from the options hash"` style cases

Per the repo convention (and the precedent of the closed
`move-cpk-polymorphic-through-test-to-trails-file` and
`bigint-roundtrip-rename-to-trails-test-file` stories), TS-only extras belong in
a sibling `*.trails.test.ts`, not in the Rails-named file — otherwise they
inflate the "extra (TS only)" column of `pnpm parity:test` against a file whose
Rails counterpart has one test, and a reader cannot tell which cases are ported
and which are invented.

The sqlite3 and abstract-mysql-adapter statement-pool files are ALREADY
`.trails.test.ts` and are fine; only the postgresql one is misfiled.

## Converged shape

Move the trails-only cases out of
`packages/activerecord/src/adapters/postgresql/statement-pool.test.ts` into
`packages/activerecord/src/adapters/postgresql/statement-pool.trails.test.ts`,
leaving the Rails-matched `cache is per pid` case behind. Do NOT reword any
test name in the move — `parity:test` matches on them.

## Acceptance criteria

- [ ] The Rails-mapped file contains only cases with a counterpart in
      `statement_pool_test.rb`.
- [ ] Trails-only cases live in the `.trails.test.ts` sibling, names verbatim.
- [ ] `pnpm parity:test` delta non-negative.
- [ ] PG adapter tests still pass.
