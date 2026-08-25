---
title: "mysql2/sqlite3 carry a materialized-only inTransaction getter Rails defines only on PG"
status: draft
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`in_transaction?` exists on exactly one Rails adapter: it is private on
`PostgreSQLAdapter`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:908-910`,
`open_transactions > 0`). `Mysql2Adapter` and `SQLite3Adapter` have no such
method — the cross-adapter question is `transaction_open?`
(`abstract/database_statements.rb:379-381`, `current_transaction.open?`).

trails has a public `get inTransaction()` on all three adapters, and on the two
non-PG ones it returns the private materialized-BEGIN flag `_inTransaction`:

- `connection-adapters/mysql2-adapter.ts:1453`
- `connection-adapters/sqlite3-adapter.ts:1121`

So a Rails-named accessor sits on two adapters Rails never gives it to, and it
answers a _different_ question than the name implies — a lazy (un-materialized)
frame is open but `inTransaction` is false.

[[pg-in-transaction-accessor-diverges-from-rails]] (PR #7049) converged the PG
one to Rails' body and repointed its cross-adapter reader
(`transactions.ts`, `with_transaction_returning_status`) onto the already-ported
`isTransactionOpen()` per `transactions.rb:411`. These two were out of that
story's scope and still carry the old shape.

Remaining readers of the two non-PG getters: `adapter.test.ts:110` (the SQLite
arm of the `raw_transaction_open?` helper) and
`test-fixtures/with-transactional-fixtures.trails.test.ts:210,237`.

## Converged shape

Delete both getters. The materialized flag stays private (it is a
physical-BEGIN marker with real jobs — the RETURNING savepoint wrap, the
`CREATE INDEX CONCURRENTLY` guard on PG's side), and the four test readers move
to `isTransactionOpen()` (`transaction_open?`) or to `openTransactions`,
whichever each one actually means. Do NOT give the two adapters a Rails-shaped
`in_transaction?` — Rails does not define one for them.

## Acceptance criteria

- [ ] `get inTransaction` exists only on `PostgreSQLAdapter`.
- [ ] The remaining readers assert what they mean and stay green on all lanes.
- [ ] `pnpm parity:api:extra --package activerecord` shows the two names gone.
