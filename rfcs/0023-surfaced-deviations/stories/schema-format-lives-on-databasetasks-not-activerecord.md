---
title: "ActiveRecord.schema_format lives on DatabaseTasks in trails"
status: draft
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
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

Rails keeps the schema format on the framework namespace:
`ActiveRecord.schema_format` (`vendor/rails/activerecord/lib/active_record.rb`,
`singleton_class.attr_accessor :schema_format`), and every reader names it that
way — `Tasks::DatabaseTasks.load_schema(db_config, format = ActiveRecord.schema_format)`
(`tasks/database_tasks.rb:376`), `reconstruct_from_schema` (`:413`),
`dump_schema` (`:431`), `Migration.any_schema_needs_update?`
(`migration.rb:747-751`), and `ENV.fetch("SCHEMA_FORMAT", ActiveRecord.schema_format)`
(`railties/databases.rake:537`).

trails has no `ActiveRecord.schemaFormat`. The setting lives on
`DatabaseTasks.schemaFormat` (`packages/activerecord/src/tasks/database-tasks.ts:123`)
— a trails invention: Rails' `DatabaseTasks` defines no such accessor, it only
_reads_ `ActiveRecord.schema_format` as a default argument.

Surfaced while porting `any_schema_needs_update?` / `load_schema!` in #6168,
where both new bodies had to read `databaseTasks.schemaFormat` and cite the
divergence at the call site (`migration.ts`, `anySchemaNeedsUpdate` and
`loadSchemaBang` JSDoc).

## Converged shape

`schemaFormat` moves to the `ActiveRecord` config object
(`packages/activerecord/src/ar-config.ts`, next to `maintainTestSchema`), and
`DatabaseTasks`' default arguments read it from there, exactly as
`database_tasks.rb:376` does. Every `DatabaseTasks.schemaFormat` reader is
updated; the two call-site notes in `migration.ts` are deleted.

## Acceptance criteria

- `ActiveRecord.schemaFormat` exists and is the single source of the setting.
- `DatabaseTasks.loadSchema` / `schemaUpToDate` / `reconstructFromSchema` /
  `dumpSchema` default their `format` parameter to `ActiveRecord.schemaFormat`,
  per `database_tasks.rb:376,413,431`.
- `DatabaseTasks.schemaFormat` is removed, not aliased — a delegating alias
  keeps the invented name alive and is not a convergence.
- The `ActiveRecord.schemaFormat` deviation notes in `migration.ts`
  (`anySchemaNeedsUpdate`, `loadSchemaBang`) are deleted.
