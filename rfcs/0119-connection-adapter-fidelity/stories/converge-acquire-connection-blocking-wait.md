---
title: "acquire_connection owns the blocking queue wait, as connection_pool.rb:862 does"
status: draft
updated: 2026-08-13
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ConnectionPool#checkout`
(`packages/activerecord/src/connection-adapters/abstract/connection-pool.ts`)
now follows Rails' guard clause after PR #6464:

    # activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb:548
    return checkout_and_verify(acquire_connection(checkout_timeout)) unless @pinned_connection

but the guard's body is INLINED rather than delegated: the port calls
`_tryAcquire()`, then falls through to its own `_available.poll(t)` wait, the
discard checks and `_checkedOut.add(c)`. Rails puts all of that on
`#acquire_connection(checkout_timeout)`
(connection_pool.rb:862), whose job is exactly "return a connection, blocking on
the queue up to the timeout".

trails already HAS an `_acquireConnection()` — but it is a different method: it
resolves the pinned connection, tries a non-blocking acquire and raises
`ConnectionTimeoutError` immediately. It takes no timeout and never waits, so
`checkout` cannot call it. Two methods share Rails' one name and neither matches
its body.

## Converged shape

Move the blocking acquire into `_acquireConnection(checkoutTimeout)` so it is
Rails' `acquire_connection` — poll the queue, wait up to the timeout, raise
`ConnectionTimeoutError` on expiry — and reduce `checkout`'s guard clause to the
one-line delegation connection_pool.rb:548 is. Reconcile or rename the existing
non-blocking helper; Rails has no second method here.

## Acceptance criteria

- [ ] `_acquireConnection(checkoutTimeout)` owns the queue wait and the timeout
      raise, mirroring connection_pool.rb:862.
- [ ] `checkout`'s unpinned arm is `return checkoutAndVerify(await this._acquireConnection(timeout))`.
- [ ] No second same-named helper with a different body survives.
- [ ] `pnpm parity:api:calls` / `:args` do not regress; connection-pool suites
      green on all three adapters.
