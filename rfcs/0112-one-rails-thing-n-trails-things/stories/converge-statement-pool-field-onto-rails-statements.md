---
title: "Rename the adapter statement-pool field to _statements and drop _statementPoolForTest"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 90
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #5577 converged the three adapters onto a single statement-pool field
name, `_statementPool` (`postgresql-adapter.ts:476`,
`sqlite3-adapter.ts:318`, `mysql2-adapter.ts:274`). That fixed the
mysql2-only `_stmtPool` divergence, but the surviving name is still not
Rails'. Rails stores the pool in `@statements`, assigned once in
`abstract_adapter.rb:156` (`@statements = build_statement_pool`) and read
under that name by every adapter — `mysql2/database_statements.rb:63`,
`sqlite3/database_statements.rb:82`, `postgresql_adapter.rb:922-933` — and
by `abstract_adapter.rb:739-746` (`clear_cache!`).

Two deviations remain:

1. The field is `_statementPool`, not `_statements`.
2. Each adapter carries a trails-invented `_statementPoolForTest()`
   accessor (`postgresql-adapter.ts:3159`, `mysql2-adapter.ts:382`, and the
   sqlite3 equivalent). Rails has no such method — tests reach the pool with
   `instance_variable_get(:@statements)` (`bind_parameter_test.rb:273-274`).
   Callers today: `hot-compatibility.test.ts:16`,
   `adapters/abstract-mysql-adapter/statement-pool.trails.test.ts` (8 sites),
   `adapters/postgresql/statement-pool.test.ts`.

## Acceptance criteria

- The pool field is named `_statements` on all three adapters, matching
  Rails' `@statements`.
- `_statementPoolForTest()` is deleted; its callers read the field directly,
  the way `bind-parameter.test.ts`'s `statementCacheKeys` already does.
- No adapter branching is reintroduced in any test helper.
- `parity:api` / `parity:test` deltas stay non-negative; sqlite3,
  postgresql, and the `ARCONN=mysql2 MYSQL_PREPARED_STATEMENTS=1` lanes stay
  green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
