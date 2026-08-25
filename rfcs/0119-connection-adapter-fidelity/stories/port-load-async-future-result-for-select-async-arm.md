---
title: "port load_async/FutureResult so DatabaseStatements#select takes its async arm"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 500
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`DatabaseStatements#select` in trails
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts`,
`select`) accepts an `async` option in `_options` and ignores it, delegating
straight to `internalExecQuery`.

Rails' `#select`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:671`)
has two arms:

- `if async && async_enabled?` — raise
  `AsynchronousQueryInsideTransactionError` when
  `current_transaction.joinable?`, otherwise schedule/execute a `FutureResult`.
- otherwise — `internal_exec_query(...)`.

trails ports only the second arm. The guard cannot be wired on its own: there
is no `FutureResult`, no `async_enabled?`, and no async executor.
`Relation#loadAsync` (`packages/activerecord/src/relation.ts`) is a
promise-stashing shim, not Rails' `load_async`, so `async` never reaches
`select` as true.

The predecessor story
`wire-asynchronous-query-inside-transaction-error-with-load-async` was closed
as not actionable standalone for exactly this reason (PR #6283 removed its
now-stale citation from the call site). Nothing currently owns the
infrastructure, which is why this story exists.

## Acceptance criteria

- `FutureResult` and `async_enabled?` are ported far enough that `select`'s
  `async` argument is real.
- `select` takes both Rails arms in Rails' order, raising
  `AsynchronousQueryInsideTransactionError` under Rails' exact condition
  (`async && async_enabled? && current_transaction.joinable?`) with Rails'
  message.
- The `@internal` note at the `select` call site explaining the unwired guard
  is deleted, not reworded.
