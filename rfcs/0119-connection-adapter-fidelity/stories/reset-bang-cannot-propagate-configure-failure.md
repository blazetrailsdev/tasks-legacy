---
title: "AbstractAdapter#resetBang cannot propagate attempt_configure_connection's raise"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `AbstractAdapter#resetBang` cannot propagate `attempt_configure_connection`'s raise

## Context

Surfaced in PR #7038 (RFC 0119,
`abstract-reset-bang-missing-attempt-configure-connection`).

Rails' `reset!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:725-731`)
ends in `attempt_configure_connection`, which disconnects and RE-RAISES
(`:1216-1221`):

```ruby
def attempt_configure_connection
  configure_connection
rescue
  disconnect!
  raise
end
```

so a pool reset whose reconfigure fails raises out of `reset!` into the caller.

trails' `resetBang`
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`) is `void`
and synchronous by the pool's contract, so the async hop cannot be awaited and
the raise has nowhere to go. PR #7038 routed the rejection to
`ActiveSupport.errorReporter.report(error, { handled: true })` — better than the
silent `.catch(() => {})` it replaced, but still not Rails: the caller of
`reset!` does not see the failure, and only the NEXT lease surfaces it (the
connection is already disconnected by then, so it reconnects and raises there).

The PG override has the same shape one level down: its whole locked body is
`void this.lock.synchronize(...).catch(() => {})`
(`postgresql-adapter.ts`, `override resetBang`).

## Converged shape

`reset!` is async in trails terms, so `resetBang` returns the promise and its
callers await it, letting `attemptConfigureConnection`'s rejection propagate to
whoever asked for the reset — the same convergence
[[project_pool_async_sync_surface_convergence]] tracks for the rest of the pool
surface. The error reporter hop then goes away rather than standing in for the
raise.

## Acceptance criteria

- [ ] `AbstractAdapter#resetBang` propagates a failed
      `attemptConfigureConnection` to its caller instead of reporting it.
- [ ] The PG override's `.catch(() => {})` around the locked body goes with it.
- [ ] Every `resetBang` caller awaits (or deliberately forwards) the result.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
