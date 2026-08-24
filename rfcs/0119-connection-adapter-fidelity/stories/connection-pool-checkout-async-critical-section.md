---
title: "connection-pool-checkout-async-critical-section"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
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

`ConnectionPool#checkout` is async in trails (RFC 0023
`converge-connection-pool-checkout-lease-async`, PR #4466) and mutates shared
pool state AFTER an await, so two concurrent checkouts can interleave in
exactly the window Rails closes with a mutex:

- `packages/activerecord/src/connection-adapters/abstract/connection-pool.ts`
  — the pinned arm awaits `verifyBang()` and only then pushes onto
  `this._connections`; the waiting arm awaits `this._available.poll(t)` and
  only then does `this._checkedOut.add(c)`.
- Rails wraps both arms in a mutex:
  `activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb:547-579`
  (`@pinned_connection.lock.synchronize` on the pinned branch, `synchronize`
  around `try_to_checkout_new_connection`).

This is NOT the general "trails is single-threaded so a mutex has no analogue"
case that PR #6364 suppressed via `NO_JS_CALL_FORM` — that argument holds only
for a body with no yield point, or one whose mutations all precede its first
await (`discardPoolBang`, `_disconnect`, `_flush`). `checkout` is the one pool
method where the interleaving is reachable, and the call gate can no longer
see it: `synchronize` is name-suppressed for the whole repo, so no baseline row
or ratchet flag will ever point at this again. This story is the record.

Sibling methods to re-check with the same test (mutation before first await?):
`ConnectionPool#disconnect`, `#discardBang`, `#clearReloadableConnections`,
`#flush` were each read during #6364 and DO hold the invariant today — they
mutate in a synchronous core and only then await the drains that core returns.

## Acceptance criteria

- Decide and record whether the interleaving is actually reachable in trails'
  concurrency model — a failing test that checks out concurrently and observes
  a duplicated `_connections` entry or an over-subscribed `_checkedOut` is the
  evidence; if it is NOT reachable, say why with the `file:line` that makes it
  unreachable and close.
- If reachable: serialize the async section (an await-queue over the pool's
  critical section is the shape Rails' mutex has), keeping the Rails method
  names and the per-checkout verify/self-heal semantics.
- Do not "converge back" to a synchronous `checkout`: `verifyBang` is async in
  trails by design (abstract_adapter.rb:759 is sync; the trails port issues a
  real backend liveness round-trip).
- Add the regression test to the pool's own test file; it must fail on
  baseline.
