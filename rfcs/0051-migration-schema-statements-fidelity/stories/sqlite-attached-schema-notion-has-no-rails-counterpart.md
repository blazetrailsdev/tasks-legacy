---
title: "Decide the SQLite3 ATTACHed-schema notion: _splitTableName and the quotedIndexNameAndTable seam have no Rails counterpart"
status: claimed
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: "2026-08-25T15:46:30Z"
assignee: "sqlite-attached-schema-notion-has-no-rails-counterpart"
blocked-by: null
closed-reason: null
---

## Context

trails' SQLite3 adapter carries an ATTACHed-schema notion that Rails has
nowhere in `sqlite3_adapter.rb`: `SQLite3Adapter#_splitTableName`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:1589`)
splits an `aux.posts` table name into `{ schema, bare }`, and ten call sites
consume it — `indexName` (:1618), `:1656`, `:1856`, `:2109`, `:2152`, `:2371`,
`:2430`, and `copyTableIndexes` (:2648-2649).

Nothing in Rails' SQLite3 adapter takes a qualified table name. `indexes`,
`alter_table`, `copy_table`, `copy_table_indexes`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb`)
all treat `table_name` as a bare identifier, and the PRAGMA calls are built
without a schema qualifier.

The notion forces one Rails-less seam in an otherwise mirrored body:
`SchemaCreation#quotedIndexNameAndTable`
(`packages/activerecord/src/connection-adapters/abstract/schema-creation.ts:339`),
extracted from the inline fragment at
`abstract/schema_creation.rb:115` purely so SQLite3 can override it
(`sqlite3/schema-creation.ts`) — SQLite spells an ATTACHed schema on the INDEX
name, not the table, and qualifying the table is a syntax error.

`collapse-the-sqlite-create-index-visitor-fork` (PR #7033) shipped the cheaper
of that story's two candidates — reduce the fork to the one differing clause —
and explicitly left the second open: "establish whether the ATTACHed-schema
support is wanted at all. If the qualified-table paths were dropped, both the
override and `_splitTableName` go with them." This story is that question.

## Converged shape

Establish whether any trails consumer actually passes a schema-qualified table
name into the SQLite3 adapter (the canonical schema and Rails' own test suite
do not — see `sqlite3-introspection.trails.test.ts`, which is the only
exerciser). If nothing outside that test needs it:

- delete `_splitTableName` and unqualify its ten call sites, restoring each to
  the Rails body it diverged from;
- delete `SQLite3::SchemaCreation#quotedIndexNameAndTable`, and with it the
  `quotedIndexNameAndTable` seam on the abstract visitor, re-inlining the
  fragment at `abstract/schema_creation.rb:115` so
  `visit_CreateIndexDefinition` is a byte-for-byte mirror again;
- delete the ATTACHed-schema cases from
  `sqlite3-introspection.trails.test.ts`.

If a consumer does need it, the finding is that the notion has no Rails
counterpart and needs a `@noRailsEquivalent` receipt at `_splitTableName`
naming that consumer — not ten silently-diverged bodies.

Note this is the LAST seam standing: the whole-body visitor fork is already
gone, and `copy_table_indexes` already makes the single unconditional
`add_index` call Rails makes (`sqlite3_adapter.rb:674`).

## Acceptance criteria

- [ ] A decision is recorded, with the consumer named if the notion stays.
- [ ] If dropped: `_splitTableName`, the SQLite3 `quotedIndexNameAndTable`
      override, and the abstract seam are all gone, and
      `visit_CreateIndexDefinition` mirrors `schema_creation.rb:106-123` with
      no extraction.
- [ ] `copy_table_indexes` keeps its single unconditional `add_index` call.
- [ ] `pnpm parity:api:extra --package activerecord` novel count does not
      increase; SQLite, PostgreSQL and MySQL/MariaDB lanes green.
