---
title: "columnsHash() does not re-reflect after in-transaction DDL, blocking citext change_table assertion"
status: in-progress
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: 7044
claim: "2026-08-25T15:46:30Z"
assignee: "sqlite-attached-schema-notion-has-no-rails-counterpart"
blocked-by: null
closed-reason: null
---

## Context

Found while converging `change_table` in PR #5628
(`change-table-recorder-and-adapter-direct-yield-adapter-table`).

`packages/activerecord/src/adapters/postgresql/citext.test.ts` "change table
supports json" carries a `TODO` that drops the middle of Rails'
`test_change_table_supports_json`
(`vendor/rails/activerecord/test/cases/adapters/postgresql/citext_test.rb:41-54`):

```ruby
@connection.change_table("citexts") { |t| t.citext "username" }
Citext.reset_column_information
column = Citext.columns_hash["username"]
assert_equal :citext, column.type
```

The `t.citext "username"` half is converged and green. The two assertion lines
are still commented out. The `TODO` blames
`InstrumentationAlreadyStartedError` in the PG driver, which is **stale/wrong** —
I restored the assertion and ran it against a real PG instance on #5628, and it
fails differently:

```text
TypeError: Cannot read properties of undefined (reading 'type')
  at citext.test.ts:69  →  Citext.columnsHash()["username"]
```

`Citext.resetColumnInformation()` runs, but the synchronous `columnsHash()` has
no `username` entry for a column added by DDL inside an open
`connection.transaction()`. So this is a **schema-reflection** deviation
(sync `columnsHash()` cannot re-reflect after in-transaction DDL), not a driver
instrumentation bug.

## Acceptance criteria

- `columnsHash()` re-reflects after `resetColumnInformation()` inside an open
  transaction, matching `reset_column_information` + `columns_hash`
  (`activerecord/lib/active_record/model_schema.rb`).
- `citext.test.ts` "change table supports json" restores
  `column = Citext.columns_hash["username"]` / `assert_equal :citext,
column.type` and drops the `TODO` (the stale
  `InstrumentationAlreadyStartedError` reason must not survive either way).
- The restored assertion is verified against a real PostgreSQL lane, not
  sqlite3 — the test is PG-only.
- `parity:api` / `parity:test` deltas non-negative.
- If the synchronous accessor genuinely cannot re-reflect, that is a TypeScript
  language wall: `pnpm tasks block` naming it, rather than documenting the
  deviation and closing.
