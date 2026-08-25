---
title: "Centralize the six _isConnectionError discard catches in withRawConnection"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

## Context

Surfaced while closing `pg-exec-rollback-db-transaction-body-deviation`.
PR #6274 audited the three additions the port had wrapped around
Rails' two-line `exec_rollback_db_transaction` and deleted two of them — among
them an `_isConnectionError(e)` → `_discardRawConnection()` catch. Removing it
left 160 PG test files / 2687 tests green, and a new trails test
(`postgresql-adapter.exec-rollback-db-transaction.trails.test.ts`) severs the
transaction's socket and rolls back to show the query path recovers on its own.

That deletion converged **one** instance. The same shape remains at six other
sites in `postgresql-adapter.ts`, each a hand-rolled `catch` that inspects
`PostgreSQLAdapter._isConnectionError(...)` and tears the client down:

- `:1964` — `beginDbTransaction`
- `:1996` — `commit` (direct path)
- `:2037` — `rollback` (direct path; swallows and returns)
- `:2241` — `beginIsolatedDbTransaction`
- `:3235`
- `:4452`

(plus `:2841`, a non-conditional call inside `disconnectBang`, which is fine and
not in scope.)

Rails has none of these. Connection invalidation happens **once**, inside
`with_raw_connection`'s rescue
(`activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:1015-1032`):

```ruby
begin
  yield @raw_connection
rescue => original_exception
  translated_exception = translate_exception_class(original_exception, nil, nil)
  invalidate_transaction(translated_exception)
  ...
  elsif reconnectable && retryable_connection_error?(translated_exception)
    reconnect!(restore_transactions: true)
```

so every adapter method that runs a query inherits invalidation, retry and
reconnect from the one wrapper rather than restating it. The trails sites are a
compensation for the single persistent `pg.Client` (there is no pool
checkout/release to discard on), but they duplicate a policy Rails already
centralises, and they diverge from each other — `:2037` swallows the error and
returns, the rest rethrow.

Note the MySQL side already did this: `phase2-route-data-path-through-withrawconnection`
(RFC 0021) routed the data path through `withRawConnection`, and
`mysql2-adapter.ts:639` uses it today. PG has a `withRawConnection` available on
the same seam but the transaction-control and DDL bodies above never adopted it.

## Converged shape

Delete the six conditional `_discardRawConnection()` catches and let
`withRawConnection`'s rescue own invalidation, matching
`abstract_adapter.rb:1015-1032`. Each body then reduces to its Rails
counterpart's statements. Where a site genuinely needs teardown that Rails gets
from `reconnect!(restore_transactions: true)`, that belongs in the shared
wrapper, not restated per call site.

Belongs to the RFC 0013 pg-rawconn-convergence family, which is closed — filed
here as a surfaced deviation instead.

Sequence after `pg-rollback-direct-path-skips-cancel-any-running-query`, which
rewrites the `:1996` / `:2037` bodies (the trails-only direct `commit()` /
`rollback()` pair) and would otherwise conflict.

## Acceptance criteria

- [ ] No `_isConnectionError` → `_discardRawConnection()` catch remains in an
      adapter method body; the conditional teardown lives in one place.
- [ ] The surviving bodies match their Rails counterparts statement for
      statement, with any residual deviation cited at the call site.
- [ ] `:2037`'s swallow-and-return is gone — no site silently absorbs a
      connection error the others rethrow.
- [ ] Green on PG; `postgresql-adapter.exec-rollback-db-transaction.trails.test.ts`
      (the severed-socket recovery pin) still passes.
