---
title: "converge-current-transaction-scope-onto-connection"
status: ready
updated: 2026-08-24
rfc: "0096-naming-identifier-burndown"
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

Split out of `converge-transactions-splat-and-transaction-receiver` (RFC 0096
wave-5 residual arg-shape findings).

Rails' `Transactions#with_transaction_returning_status`
(`activerecord/lib/active_record/transactions.rb:408-421`) opens the transaction
on the yielded connection: `connection.transaction do ... end`.

trails calls the ClassMethods-level `transaction(modelClass, fn)`
(`packages/activerecord/src/transactions.ts:87`) instead, because that is the
only place that installs the `ar_current_transaction` execution-state scope
(`transactions.ts:120`, `IsolatedExecutionState.scope(CURRENT_TRANSACTION_KEY,
…)`). The connection-level `DatabaseStatements#transaction`
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:568`)
does not, so calling it directly leaves `currentTransaction()` null inside the
block — `addToTransaction` / `rememberTransactionRecordState` depend on it.

The call site carries a `@missingRailsArgs connection.transaction —
CONVERGEABLE` receipt pointing here.

## Acceptance criteria

1. The `ar_current_transaction` scope is installed by the connection-level
   `transaction` (`database-statements.ts:568`), so `connection.transaction`
   alone gives Rails' semantics.
2. `with_transaction_returning_status` calls the yielded `connection`'s
   `transaction`, as `transactions.rb:413` does, and the `@missingRailsArgs`
   receipt on it is deleted.
3. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green; the
   transaction suites pass on every adapter lane.
