---
title: "dump_schema creates the schema file's dirname, where Rails creates db_dir"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# dump_schema creates the dirname of the schema file, where Rails creates db_dir

## Context

Surfaced converging the `tasks/*` call-set rows in #6664 while hoisting the
`mkdir` above the format branch to match Rails' statement order. The hoist
landed; the argument divergence did not, and is not covered by any call-set row
(the comparator credits the call, not its receiver path).

Rails, `vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:431-437`:

```ruby
def dump_schema(db_config, format = ActiveRecord.schema_format) # :nodoc:
  return unless db_config.schema_dump

  require "active_record/schema_dumper"
  filename = schema_dump_path(db_config, format)
  return unless filename

  FileUtils.mkdir_p(db_dir)
```

Rails creates `db_dir` — the configured schema directory — unconditionally.
trails' `DatabaseTasks.dumpSchema`
(`packages/activerecord/src/tasks/database-tasks.ts`) creates
`path.dirname(filename)` instead, after resolving `filename` through
`_resolveSchemaPath`. The two agree whenever the dump lands directly in
`db_dir`, and diverge as soon as `ENV["SCHEMA"]` or a per-config
`schema_dump` points somewhere else: Rails still creates `db_dir` (and fails on
the missing target dir), trails creates the target dir and never touches
`db_dir`.

## Converged shape

`fs.mkdirSync(this.dbDir, { recursive: true })` at the same point in the body,
matching `:437`. Check whether any test depends on the target directory being
created for it before flipping — if one does, that test is asserting trails
behaviour Rails does not have and should be looked at alongside.

## Acceptance criteria

- [ ] `dumpSchema` creates `dbDir`, not the schema file's dirname.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green — `dump_schema` runs on
      every structure/schema dump path.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` still green.
