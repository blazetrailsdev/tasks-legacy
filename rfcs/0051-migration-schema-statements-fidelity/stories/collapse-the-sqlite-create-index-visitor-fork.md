---
title: "Collapse the SQLite visit_CreateIndexDefinition fork back onto the abstract body"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
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

PR 6338 added `SQLite3::SchemaCreation#visitCreateIndexDefinition`
(`packages/activerecord/src/connection-adapters/sqlite3/schema-creation.ts`),
an override Rails does not have. It is a near-copy of the abstract body
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_creation.rb`,
`visit_CreateIndexDefinition`) with one line changed: the qualifier moves off
the table and onto the index name, because SQLite spells an ATTACHed schema as
`CREATE INDEX "aux"."by_name" ON "widgets" (...)` and qualifying the table is a
syntax error.

The override bought a bigger convergence — `copy_table_indexes` is now the
single unconditional `add_index` call Rails has
(`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:674`),
with its hand-built `CREATE INDEX` string and branch-local `MISSING RAILS CALL`
comment deleted — so it is a net reduction in deviation. It is still a forked
copy of a mirrored body, and every future edit to the abstract
`visit_CreateIndexDefinition` (a new capability arm, a reordered clause) has to
be mirrored into it by hand or the two silently drift.

Rails has no ATTACHed-schema notion anywhere in this adapter, so upstream will
never grow the seam; the fork exists only because trails does
(`SQLite3Adapter#_splitTableName`, reached through `alter_table` / `copy_table`
with an `aux.posts` name).

## Converged shape

Two candidates, cheapest first:

- Reduce the fork to the one differing clause: the abstract body grows a
  protected seam for the `"<name> ON <table>"` fragment that SQLite3 alone
  overrides, so the arm order and every capability gate live in one place. That
  is a Rails-less seam too, but a one-line one rather than a whole-body copy.
- Or establish whether the ATTACHed-schema support is wanted at all. If the
  qualified-table paths were dropped, both the override and `_splitTableName`
  go with them.

Either way the goal is one body, not two that must be kept in step by hand.

## Acceptance criteria

- [ ] The SQLite3 override is no longer a whole-body copy of the abstract
      `visit_CreateIndexDefinition`, OR the ATTACHed-schema notion is gone and
      the override with it.
- [ ] `copy_table_indexes` keeps its single unconditional `add_index` call.
- [ ] The ATTACHed-schema case in
      `packages/activerecord/src/connection-adapters/sqlite3-introspection.test.ts`
      stays green.
