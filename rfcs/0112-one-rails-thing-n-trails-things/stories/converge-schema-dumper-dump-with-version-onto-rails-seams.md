---
title: "Remove invented SchemaDumper.dumpWithVersion; use dump_schema_information + dump header"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
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

`SchemaDumper.dumpWithVersion(adapter)`
(`packages/activerecord/src/schema-dumper.ts:527-541`) has no Rails counterpart.
Rails splits this into two independent things:

- `ActiveRecord::ConnectionAdapters::SchemaStatements#dump_schema_information`
  (`activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1355-1358`)
  — emits the `INSERT INTO schema_migrations ...` statements only. Already
  ported faithfully at
  `packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1681`.
- `ActiveRecord::SchemaDumper.dump` — emits the schema body. The migration
  version reaches the output through the `ActiveRecord::Schema[<version>].define`
  header (`activerecord/lib/active_record/schema_dumper.rb#define_params` /
  `header`), not through a `// Schema version: N` comment line.

`dumpWithVersion` fuses the two: it reads `schema_migrations`, then does a
full-database dump, then prefixes a trails-invented `// Schema version: N`
comment. That fusion is also a real cost — it enumerates and dumps every table,
which is what timed out `dump schema information with empty versions` on the
canonical PG schema and red-ed `Active Record PostgreSQL Tests (2)` on main
@eb32c962 (fixed by PR #6081, which moved the two `dump schema information ...`
cases onto `dumpSchemaInformation`).

PR #6081 left one caller behind: the `schema dump include migration version`
case in `packages/activerecord/src/schema-dumper.test.ts` still drives
`dumpWithVersion` and asserts on the invented `"Schema version: ..."` /
`"defineSchema"` strings. Rails' `test_schema_dump_include_migration_version`
(`activerecord/test/cases/schema_dumper_test.rb:44-47`) asserts
`%r{ActiveRecord::Schema\[#{ActiveRecord::Migration.current_version}\]\.define}`
against `standard_dump`. It carries the same full-dump cost and is a live
timeout risk on the PG lane.

## Acceptance criteria

- `SchemaDumper.dumpWithVersion` is removed; its two responsibilities are served
  by the already-ported `dumpSchemaInformation` and by `SchemaDumper.dump`
  emitting the version in its header the way Rails does.
- The `// Schema version: N` comment prefix is gone — the version reaches the
  output via the schema-definition header, per `schema_dumper.rb`'s `header` /
  `define_params`.
- `schema dump include migration version` asserts against the dump header rather
  than the invented `"Schema version: ..."` string.
- All in-repo callers of `dumpWithVersion` are migrated; no `@noRailsEquivalent`
  tag is added to keep it.

Note: `ActiveRecord::Migration.current_version` / the `Schema[x.y]` version
bracket is out of scope for trails, so the header assertion should target the
`define` call shape trails actually emits — the point of this story is deleting
the invented `dumpWithVersion` seam and the invented comment line, not porting
migration version compatibility.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
