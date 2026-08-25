---
title: "SQLite3 DatabaseStatements has two homes for one Rails method"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# SQLite3 `DatabaseStatements` has two homes for one Rails method

## Context

Surfaced while converging the SQLite3 transaction-control rows in PR #6718
(RFC 0106 wave 4b).

`activerecord/lib/active_record/connection_adapters/sqlite3/database_statements.rb`
defines `begin_db_transaction`, `commit_db_transaction`,
`exec_rollback_db_transaction`, `reset_isolation_level`,
`internal_begin_transaction`, `execute` and `default_insert_value` once. trails
has each of them **twice**:

- `packages/activerecord/src/connection-adapters/sqlite3/database-statements.ts`
  — the file that matches the Rails path, and the one `parity:api` scores.
- `packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:780-980`
  — a second, independent implementation on the adapter class.

Only the adapter copy runs. `sqlite3-adapter.ts` imports seven names from
`sqlite3/database-statements.js` (`returningColumnValues`,
`buildTruncateStatement`, `executeBatch`, `castResult`, `affectedRows`,
`performQuery`, `highPrecisionCurrentTimestamp`) and none of the transaction
ones, so the copies in the Rails-matched file have no caller.

This is not cosmetic. PR #6718 converged the dead copies to
`internalExecute(sql, "TRANSACTION", { allowRetry: true, materializeTransactions: false })`
to match `database_statements.rb:36-41,59-60,71-74` and retire four call-set
rows — a fix to code nothing executes. The live copy on the adapter is the one
whose fidelity actually matters, and the two can drift apart silently: nothing
type-checks them against each other and no test covers the dead one.

## Converged shape

One Rails method, one TS home, in the file that matches the Rails path. Move the
live bodies into `sqlite3/database-statements.ts` as `this`-typed functions (the
mixin idiom in CLAUDE.md, as `internalBeginTransaction` there already is), assign
them onto `SQLite3Adapter` the way the other seven are wired, and delete the
adapter-class copies. Keep the Rails names and the Rails argument lists.

Check before moving whether the adapter copies carry behaviour the dead ones
lack (the `_previousReadUncommitted` bookkeeping and the `withRawConnection`
`ensure` semantics around `internalExecute` are the likely spots) — the move must
preserve the LIVE behaviour, then converge it to Rails, not silently swap in the
dead body.

## Acceptance criteria

- [ ] Each of the seven methods exists once, in
      `connection-adapters/sqlite3/database-statements.ts`.
- [ ] `sqlite3-adapter.ts` wires them the way it wires the existing seven; no
      duplicate body remains on the class.
- [ ] `pnpm parity:api` delta non-negative — the methods must still score
      against `sqlite3/database_statements.rb`.
- [ ] `pnpm parity:api:calls` / `:args` green.
- [ ] SQLite lane green, including the isolation-level and savepoint tests that
      exercise the live path today.
