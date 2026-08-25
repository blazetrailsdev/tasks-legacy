---
title: "cacheDumpFilename adds a fourth fallback and any-casts Rails does not have"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

`DatabaseTasks.cacheDumpFilename`
(`packages/activerecord/src/tasks/database-tasks.ts:650`) does more than the
Ruby it stands in for. Rails is three `||` terms and no fallback of its own
(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:468-471`):

```ruby
def cache_dump_filename(db_config, schema_cache_path: nil)
  schema_cache_path ||
    db_config.schema_cache_path ||
    db_config.default_schema_cache_path(ActiveRecord::Tasks::DatabaseTasks.db_dir)
end
```

The port adds a fourth term — a hard-coded `` `${this.dbDir}/schema_cache.json` ``
— and reaches both `schemaCachePath` and `defaultSchemaCachePath` through
`typeof (dbConfig as any).x === "function"` probes, so a config object that
answers neither silently gets the invented default instead of the
`NoMethodError` Rails would raise. `defaultSchemaCachePath` is a plain method on
`HashConfig` (`packages/activerecord/src/database-configurations/hash-config.ts:76`,
Rails `hash_config.rb:117-123`), so the probes guard against nothing a
`DatabaseConfig` can actually be.

Surfaced while converging the five `DatabaseTasksDumpSchemaCacheTest` tests onto
this method in PR #6089 — they were the first callers to exercise it, and they
pass through the first three terms only.

Note the `.json` extension is a _separate_, deliberate trails deviation
documented at `hash-config.ts:76` (trails dumps JSON, not YAML); this story is
about the extra fallback term and the `as any` probes, not the extension.

## Converged shape

- Three terms, in Rails' order, with Rails' `||` semantics, and no fourth
  fallback.
- `dbConfig.schemaCachePath` and `dbConfig.defaultSchemaCachePath(this.dbDir)`
  called directly on the typed `DatabaseConfig`, with the `as any` casts and the
  `typeof === "function"` probes deleted.

## Acceptance criteria

- `cacheDumpFilename` has no `as any` cast and no fourth fallback term.
- The five `DatabaseTasksDumpSchemaCacheTest` tests still pass unchanged, and
  `parity:test` for `tasks/database_tasks_test.rb` stays at 77/77.
