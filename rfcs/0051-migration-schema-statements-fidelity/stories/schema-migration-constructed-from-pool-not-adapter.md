---
title: "SchemaMigration takes an adapter where Rails takes a pool"
status: closed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Premise gone on origin/main. SchemaMigration's constructor is now constructor(pool: ConnectionPool | NullPool) (schema-migration.ts:57), matching schema_migration.rb:14, and ConnectionPool exposes both accessors — get schemaMigration(): SchemaMigration (connection-pool.ts:593) and get internalMetadata(): InternalMetadata (:597), with the NullPool declare-readonly stubs at :122-123. multi-db-migrator.test.ts:73-74 now builds new SchemaMigration(adapterA.pool) / (adapterB.pool) rather than leasing a connection. Residual nit only: that beforeEach spells the pool as adapterX.pool rather than Base.connectionPool().schemaMigration — not worth a story."
---

## Context

Rails' `ActiveRecord::SchemaMigration` is constructed from a **pool**:

- `vendor/rails/activerecord/lib/active_record/schema_migration.rb:14` —
  `def initialize(pool)`
- and it is normally reached _through_ the pool, not built by hand:
  `multi_db_migrator_test.rb:26-27` uses `@pool_a.schema_migration.create_table`
  / `@pool_b.schema_migration.create_table`, and `:58-59` uses
  `pool.schema_migration.delete_all_versions`.

trails' `SchemaMigration` takes an adapter instead
(`packages/activerecord/src/schema-migration.ts:32` — `constructor(adapter: DatabaseAdapter)`),
and `ConnectionPool` exposes no `schemaMigration` accessor
(`connection-adapters/abstract/connection-pool.ts`). So a ported test cannot
spell Rails' `pool.schema_migration` and must construct
`new SchemaMigration(await Base.leaseConnection())`.

Surfaced while converging `multi-db-migrator.test.ts` onto the canonical
`Base` / `ARUnit2Model` pools (PR #5598). That PR could port
`multi_db_migrator_test.rb:71-76`'s pool-descriptor assertions, but its
`beforeEach` still leases connections and builds `SchemaMigration` by hand where
Rails reads two pools' `schema_migration`.

## Acceptance criteria

- `SchemaMigration` is constructed from a pool, per `schema_migration.rb:14`.
- `ConnectionPool` exposes `schemaMigration` (and check `internalMetadata`
  alongside it — `multi_db_migrator_test.rb:40-41` reads
  `@pool_a.internal_metadata` the same way).
- `multi-db-migrator.test.ts`'s `beforeEach` reads
  `Base.connectionPool().schemaMigration` /
  `ARUnit2Model.connectionPool().schemaMigration`, matching
  `multi_db_migrator_test.rb:22-27`.
- Existing `new SchemaMigration(adapter)` call sites are converged or bridged;
  grep first, this constructor is used well beyond the migrator tests.

## Notes

Non-trivial: `SchemaMigration` currently issues its queries straight against the
adapter it is handed, so moving to a pool means routing through the pool's lease
path. Sequenced after the pool-checkout work in RFC 0073 if that lands first.
