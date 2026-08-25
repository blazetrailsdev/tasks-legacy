---
title: "Converge unpinConnectionBang's tear-down statement order with Rails"
status: ready
updated: 2026-08-25
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

Surfaced while reviewing PR #6408 (which added the missing
`@pinned_connection.lock.synchronize` wrapper to `unpinConnectionBang` but
deliberately did not reorder the arms inside it). The reviewer independently
flagged the same divergence.

`ConnectionPool#unpin_connection!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb:340-365`)
decrements the depth and clears the pin **before** it inspects the transaction,
and does the checkin as a plain trailing statement inside the lock:

```ruby
def unpin_connection! # :nodoc:
  raise "There isn't a pinned connection #{object_id}" unless @pinned_connection

  clean = true
  @pinned_connection.lock.synchronize do
    @pinned_connections_depth -= 1
    connection = @pinned_connection
    @pinned_connection = nil if @pinned_connections_depth.zero?

    if connection.transaction_open?
      connection.rollback_transaction
    else
      # Something committed or rolled back the transaction
      clean = false
      connection.reset!
    end

    if @pinned_connection.nil?
      connection.steal!
      connection.lock_thread = nil
      checkin(connection)
    end
  end

  clean
end
```

The port (`packages/activerecord/src/connection-adapters/abstract/connection-pool.ts`,
`unpinConnectionBang`) instead runs the rollback/reset arm first and does all the
bookkeeping — `pin.depth--`, clearing `_fixturePin` / `_pinnedConnections`,
`decrementPinnedCount()`, `checkin()` — in a `finally`.

Two behavioural consequences, both observable:

1. **Rollback failure.** Rails' checkin is a normal statement, so a raising
   `rollback_transaction` propagates out with the connection NOT checked in
   (the depth is already decremented). The port's `finally` checks the
   connection in anyway. These differ exactly when the rollback raises.
2. **Ordering.** Rails clears `@pinned_connection` before the rollback runs, so
   anything re-entering during the rollback sees an already-unpinned pool. The
   port leaves the pin registered for the duration of the rollback. PR #6408's
   lock hides this from concurrent callers, but re-entrant paths on the same
   async chain (the lock is re-entrant by design) still observe the stale pin.

The convergence is mechanical: drop the `try`/`finally`, decrement and clear the
pin first, then the transaction arm, then the `pin.depth === 0` checkin — all
inside the `synchronize` block PR #6408 already added.

Note `steal!` / `lock_thread = nil` have no trails counterpart (single-threaded
runtime, no owning thread to steal from); only `checkin` survives from that arm.

## Acceptance criteria

- [ ] `unpinConnectionBang`'s body follows `connection_pool.rb:344-362` statement
      for statement: decrement + clear pin, then the `transactionOpen` /
      `resetBang` arm, then the `pin.depth === 0` checkin — no `try`/`finally`.
- [ ] A regression test covering the rollback-raises path: the connection is NOT
      checked in and the error propagates, matching Rails. Fails on the baseline.
- [ ] The existing interleaving regression test in
      `connection-pool.trails.test.ts` ("concurrent unpinConnectionBang calls do
      not interleave inside the pin tear-down") still passes.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
