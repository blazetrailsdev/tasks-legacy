---
title: "sqlite3 beginDbTransaction carries a re-entrancy guard Rails does not have"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
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

`AbstractSQLite3Adapter#beginDbTransaction`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`) opens with a
re-entrancy guard that Rails does not have:

```ts
// DEVIATION: Rails has no re-entrancy guard; the pool proxy can replay a
// begin against an already-open connection and SQLite rejects nested BEGIN.
if (this._inTransaction) return;
```

Rails' `begin_db_transaction`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3/database_statements.rb:32-34`)
is one unconditional line: `internal_begin_transaction(:immediate, nil)`. The
guard predates the current code — it arrived with #493 (route transactions
through TransactionManager) and survived PR #5913, which converged the three
transaction-entry bodies onto `internalBeginTransaction` but deliberately left
the guard in place rather than risk a behaviour change outside that story's
call-set scope.

The guard makes a second `beginDbTransaction` a silent no-op where Rails would
let SQLite raise "cannot start a transaction within a transaction". That masks
double-begin bugs in the pool/TransactionManager layer instead of surfacing them.

## Acceptance criteria

- Establish whether any live caller actually replays a begin against an
  already-open connection (instrument or assert, don't assume) — the guard's
  stated justification is unverified.
- If nothing replays: delete the guard so `beginDbTransaction` is Rails'
  single unconditional call, and confirm the sqlite3 transaction suites plus
  `packages/activerecord/src/connection-adapters/sqlite3-adapter.transactions.test.ts`
  stay green.
- If something does replay: fix the caller so it stops, then delete the guard.
  Do not ratify the deviation in place.
- No new wide-call or assertion-ratchet debt.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
