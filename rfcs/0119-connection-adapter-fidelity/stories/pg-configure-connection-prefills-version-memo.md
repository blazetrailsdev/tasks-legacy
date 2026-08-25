---
title: "PG configure_connection pre-fills the version memo instead of letting check_version fetch"
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

Rails' `PostgreSQLAdapter#configure_connection` calls `super`
(`abstract_adapter.rb:1212-1214`), whose whole body is `check_version`, and
`check_version` (`postgresql_adapter.rb:669-673`) reads `database_version` —
which is `pool.server_version(self)` (`abstract_adapter.rb:854-856`) and issues
its own round-trip on demand. Rails pre-fills nothing.

trails' `PostgreSQLAdapter#_maybeConfigureConnection`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`, around
the `_serverVersion(client)` block) instead fills `_databaseVersion` from the
client it is configuring, then calls `await this.checkVersion()` guarded on the
memo being non-null.

This was #6224's shape and was deliberately left in place by #6226 (RFC 0072
`make-version-gated-predicates-async`), which made `databaseVersion` self-fetch
and removed the equivalent pre-fills elsewhere. The reason it stayed: the pool
memo routes through `getDatabaseVersion()` -> `_acquireFreshClient()`, which can
re-enter while the connection is still being configured. So the pre-fill is a
re-entrancy guard, not the retired shape-2 scaffolding — but it is still surface
Rails does not have, and the `if (this._databaseVersion !== null)` guard means a
zero/failed probe silently skips the version floor where Rails would raise.

## Converged shape

`_maybeConfigureConnection` calls `checkVersion()` unconditionally, as Rails'
`super` does, and `checkVersion` reads `databaseVersion` on demand. Requires
establishing that the pool memo can be filled from the connection under
configuration without re-entering `_acquireFreshClient()` — most likely by
seeding the pool memo with the already-open client rather than bypassing it.

## Acceptance criteria

- [ ] `_maybeConfigureConnection` no longer pre-fills `_databaseVersion` ahead
      of the version check, or the pre-fill is shown to be the only way to avoid
      re-entrant acquire and is justified at the call site with the Rails cite.
- [ ] The `if (this._databaseVersion !== null)` guard is gone: a failed version
      probe must not silently skip the 9.3 floor.
- [ ] `postgresql-adapter.check-version.test.ts` still passes; PG lanes green.
