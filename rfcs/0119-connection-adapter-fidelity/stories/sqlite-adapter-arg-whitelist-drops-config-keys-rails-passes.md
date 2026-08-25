---
title: "buildAdapterArg's SQLite whitelist silently drops config keys Rails passes to the adapter"
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

Surfaced by a real red in PR #6826, which retired `PoolConfig#adapterFactory`
and so routed every pool through `db_config.new_connection`.

Rails does no key filtering whatsoever. `new_connection` is

    # vendor/rails/activerecord/lib/active_record/database_configurations/database_config.rb:25-27
    def new_connection
      adapter_class.new(configuration_hash)
    end

— the whole `configuration_hash` reaches the adapter, and the adapter reads
whatever keys it knows (`sqlite3_adapter.rb` reads `@config[:database]`,
`@config[:timeout]`, `DEFAULT_PRAGMAS["foreign_keys"]` at :84-85, ...).

trails' `buildAdapterArg`
(`packages/activerecord/src/connection-adapters/adapter-args.ts:150-192`)
instead **destructures a fixed whitelist** for the SQLite branch — "Keep only
the SQLite3Adapter constructor's `options` keys" — and silently drops anything
not named there.

That fails closed and silently. `foreignKeys` was missing from the list, so
`newSqlitePool(":memory:", { foreignKeys: false })` produced an adapter with
foreign keys still **on**, and `migration/foreign-key.test.ts`'s
"does not create foreign keys when bypassed by config" went red the moment the
factory bypass was removed. PR #6826 added `foreignKeys` to the whitelist as the
minimal fix, but the shape is still wrong: the next config key an adapter learns
to read will be dropped the same way, with no error at the boundary — only a
distant behavioural difference.

Note the earlier [[mysql-adapter-arg-whitelist]] was closed as "not a
Rails-fidelity divergence" on the framing of silencing mysql2 warnings. This is
the opposite direction and is a fidelity divergence: keys Rails **would** pass
are dropped.

## Converged shape

The SQLite branch stops destructuring a whitelist and forwards the configuration
the way the PG/MySQL branches already do, so a key the adapter reads cannot go
missing between `db_config` and the constructor. If a genuine reason to strip
keys survives (driver-level "unknown option" warnings), it belongs in the
adapter constructor, where Rails puts every other config interpretation — not at
a boundary Rails does not have.

Subsumed if [[delete-build-adapter-arg-once-constructors-take-config-hash]]
lands first, which deletes the whole module; this story is the standalone fix
for the latent key-dropping while that sequencing plays out.

## Acceptance criteria

- [ ] A config key an adapter reads is no longer droppable by omission from a
      list in `adapter-args.ts`.
- [ ] A regression test fails on the current baseline: a config key not in
      today's whitelist reaches the adapter.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
