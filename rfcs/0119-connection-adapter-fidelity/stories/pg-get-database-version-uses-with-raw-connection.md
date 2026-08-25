---
title: "PostgreSQLAdapter#getDatabaseVersion bypasses withRawConnection for a fresh client"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQLAdapter#getDatabaseVersion`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`, the
method whose JSDoc cites `postgresql_adapter.rb:635-643`) does not go through
`withRawConnection`. Rails' body is:

```ruby
# activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:635-643
def get_database_version # :nodoc:
  with_raw_connection do |conn|
    version = conn.server_version
    if version == 0
      raise ActiveRecord::ConnectionFailed, "Could not determine PostgreSQL version"
    end
    version
  end
end
```

trails instead takes `this._rawConnection ?? (await this._acquireFreshClient())`
and hand-rolls the dead-socket recovery (`_isConnectionError` →
`_discardRawConnection`) that `with_raw_connection`'s retry loop provides. The
stated reason at the call site is re-entrancy: the fetch is reached from
`_maybeConfigureConnection`, and `_acquireFreshClient()` would await the very
acquire it is nested inside — Ruby's `with_raw_connection` is re-entrant on
`@raw_connection`, ours is not.

This is recorded as a call-mismatch baseline row
(`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/postgresql-adapter.json`,
`get_database_version` / `with_raw_connection`) — debt, not permission.

## Converged shape

`getDatabaseVersion()` is `withRawConnection(conn => ...)`, with the zero-version
`ConnectionFailed` raise inside the block and no bespoke discard/recovery. That
requires `withRawConnection` to be re-entrant on an already-held raw connection
the way Ruby's is (return the in-hand `@raw_connection` rather than re-entering
the acquire path), which is the actual work here.

## Acceptance criteria

- [ ] `PostgreSQLAdapter#getDatabaseVersion` body is `withRawConnection` over
      the `server_version` probe and the zero-version raise, matching
      `postgresql_adapter.rb:635-643`.
- [ ] No bespoke `_isConnectionError`/`_discardRawConnection` recovery in that
      method — the lease loop owns it.
- [ ] The `get_database_version` / `with_raw_connection` baseline row is
      deleted, not reseeded.
- [ ] PG lane green, including
      `test_reconnect_after_bad_connection_on_check_version`
      (`postgresql_adapter_test.rb:98-116`), which drives this path through
      `reconnect!`.
