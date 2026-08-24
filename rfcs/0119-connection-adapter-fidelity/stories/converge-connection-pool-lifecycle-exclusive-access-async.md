---
title: "converge-connection-pool-lifecycle-exclusive-access-async"
status: draft
updated: 2026-08-03
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: ["converge-sync-connection-lease-per-checkout-verify"]
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

`converge-connection-pool-lifecycle-async` landed as #4472, but the lifecycle
path it named is still synchronous.

`packages/activerecord/src/connection-adapters/abstract/connection-pool.ts:1680-1693`
(`checkoutForExclusiveAccess`, mirroring
`ActiveRecord::ConnectionAdapters::ConnectionPool#checkout_for_exclusive_access`)
calls `pool._acquireConnection()` directly rather than `checkout`, because
`checkout` is now async (it awaits `verifyBang`) and the disconnect/discard
sweep cannot await. The comment at :1691 still says the convergence "is tracked
by `converge-connection-pool-lifecycle-async`", which is done.

Two behaviours diverge as a result: the acquired connection skips
`checkout_and_verify`'s `clean!` / query-cache wiring (Rails runs it; running it
here regressed disconnect/discard tests when last tried), and the acquisition
raises `ConnectionTimeoutError` from a different call site than Rails does.

Related: the sync-lease residual in
`converge-sync-connection-lease-per-checkout-verify` shares the same root
cause (async `checkout`), and
`project_pool_adapter_proxy_makes_sync_methods_async` records the wider shape.

## Acceptance criteria

- The disconnect/discard lifecycle sweep runs through the Rails-named
  `checkout` (or an awaited equivalent), so `checkout_and_verify`'s
  `clean!` / query-cache wiring is not skipped.
- `ExclusiveConnectionTimeoutError` is still raised with Rails' message for a
  busy pool.
- The stale `converge-connection-pool-lifecycle-async` citation at
  connection-pool.ts:1691 is removed.
- Disconnect/discard pool tests stay green.
