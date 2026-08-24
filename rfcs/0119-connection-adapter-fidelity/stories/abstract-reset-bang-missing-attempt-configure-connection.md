---
title: "AbstractAdapter#resetBang drops Rails' attempt_configure_connection"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `AbstractAdapter#reset!` (`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:725-731`) is:

```ruby
def reset!
  clear_cache!(new_connection: true)
  reset_transaction
  attempt_configure_connection
end
```

trails' `AbstractAdapter#resetBang` (`packages/activerecord/src/connection-adapters/abstract-adapter.ts:1921`) drops the third call:

```ts
resetBang(): void {
  void this.clearCacheBang({ newConnection: true });
  this.resetTransaction();
}
```

Every non-PG adapter therefore leaves the session unreconfigured after a pool
reset. PG papers over it in its own override (`postgresql-adapter.ts`
`resetBang` calls `configureConnection()` inside the locked body, landed in PR 6376), which is exactly the shape that hides the base-class gap.

Surfaced during review of PR 6376 (RFC 0061 `pg-reset-body-under-one-lock`).

## Acceptance criteria

- `AbstractAdapter#resetBang` calls `attemptConfigureConnection` after
  `resetTransaction`, in Rails' order.
- The PG override no longer needs its own configure hop if the base call covers
  it (or the override's call is the `super` dispatch, not a duplicate).
- `parity:api:calls` row for `reset!` / `attempt_configure_connection` on
  `abstract-adapter.ts` converges (deleted, not baselined).
- SQLite/MySQL/PG lanes green.
