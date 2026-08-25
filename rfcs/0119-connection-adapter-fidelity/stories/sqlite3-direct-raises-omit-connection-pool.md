---
title: "sqlite3-adapter's direct raises omit connection_pool: @pool"
status: ready
updated: 2026-08-25
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

# sqlite3-adapter's direct raises omit `connection_pool: @pool`

## Context

Surfaced converging `check_all_foreign_keys_valid!` in PR #6567 (RFC 0106
wave-3a). The body was converged onto Rails' `execute` / `blank?` / `tables`
shape, but the raise still drops one keyword Rails passes.

Rails passes the connection pool on every raise in this adapter:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:275`
  — `ActiveRecord::StatementInvalid.new("Foreign key violations found: #{tables.join(", ")}", sql: sql, connection_pool: @pool)`
- `vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:517`
  — `ActiveRecord::StatementInvalid.new("Could not find table '#{table_name}'", connection_pool: @pool)`
- `:120` — `ActiveRecord::NoDatabaseError.new(connection_pool: @pool)`

trails' two DIRECT raise sites omit it:

- `packages/activerecord/src/connection-adapters/sqlite3-adapter.ts` —
  `checkAllForeignKeysValidBang`, `new StatementInvalid(..., { sql, binds: [] })`
- same file — `tableStructure`, `new StatementInvalid(..., { sql: "", binds: [] })`

This is not a missing capability: `ActiveRecordError` already accepts and exposes
`connectionPool` (`packages/activerecord/src/errors.ts:52-61`), and the adapter's
own `translateException` passes it correctly at all five of its raise sites. Only
the two hand-written raises were missed.

Consequence: a caller rescuing either error cannot reach the pool that produced
it, where in Rails it can — which is what `ActiveRecord::StatementInvalid#pool`
consumers (connection-error reporting, pool eviction) rely on.

Note `tableStructure` also passes `sql: ""` where Rails passes no `sql:` at all;
converge that in the same pass.

## Converged shape

Both raises pass `connectionPool: this.pool` (the trails spelling of `@pool`,
already used by `translateException` in the same file), and `tableStructure`
stops inventing an empty `sql`.

## Acceptance criteria

- [ ] `checkAllForeignKeysValidBang` raises with `connectionPool`, mirroring
      sqlite3_adapter.rb:275.
- [ ] `tableStructure` raises with `connectionPool` and no invented `sql: ""`,
      mirroring sqlite3_adapter.rb:517.
- [ ] A test asserts the raised error carries the pool (the existing
      `sqlite3-adapter.check-all-foreign-keys-valid.trails.test.ts` is the natural
      home for the first one).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
