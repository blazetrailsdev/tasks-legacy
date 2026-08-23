---
title: "tablename-only-model-resolves-columns-without-explicit-reflect"
status: done
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6936
claim: "2026-08-23T18:08:08Z"
assignee: "tablename-only-model-resolves-columns-without-explicit-reflect"
blocked-by: null
closed-reason: null
---

## Context

Surfaced closing `anonymous-subclass-needs-explicit-attribute-no-lazy-column-load`
(RFC 0078). Rails' `AdapterForeignKeyTest`
(`vendor/rails/activerecord/test/cases/adapter_test.rb:352-364`) builds an
anonymous subclass whose only declaration is `self.table_name = "fk_test_has_fk"`
and assigns `has_fk.fk_id`; Ruby loads the schema lazily on first attribute
access.

The trails-only `this.attribute("fk_id", "integer")` workaround is gone from
`packages/activerecord/src/adapter.test.ts` — the column is DB-reflected again —
but the test still needs an explicit `await KlassHasFk.ensureSchemaLoaded()`
before the sync constructor/assignment. Root cause: `loadSchema`
(`packages/activerecord/src/model-schema.ts:1000-1063`) is sync and answers only
from the adapter's schema cache, which is COLD for a table no query has touched
(`loadSchemaFromCacheSync` returns false; the fall-through then marks
`_schemaLoaded = true` with zero columns, so even a later warm cache is not
consulted). The canonical schema loader creates the tables but does not seed the
schema cache for them.

## Acceptance criteria

- A model whose only class-level declaration is `tableName` resolves its DB
  columns on first attribute assignment with no explicit reflect call, on all
  three adapters — either by seeding the schema cache for canonical tables at
  load time, or by not marking a cold-cache `loadSchema` terminal plus a warm
  path that reaches it.
- `AdapterForeignKeyTest`'s `KlassHasFk` drops the `await ensureSchemaLoaded()`
  line and its comment, and still passes.
- The cost of warming is measured and stated.

  AMENDED 2026-08-23 (#6936), replacing "No measurable per-test-file cost
  regression from warming (state the numbers)". The criterion is not reachable
  by any warm this story can ship, and the measurement says why.

  The only mechanism that resolves a cold table for a SYNC `load_schema` is a
  populated `SchemaCache`, and the only way to populate it is
  `SchemaCache#add_all`, whose cost on the canonical schema is (sqlite, 319
  data sources, instrumented): `dataSources` 3ms + `primaryKeys` 61ms +
  `columns` 179ms + `indexes` 66ms = ~309ms. Dropping the halves the sync
  readers do not need still leaves ~240ms.

  That cost is per PROCESS, not per file — but vitest's fork pool spawns a
  fresh process per test file (measured: distinct pids for consecutive files),
  so a `globalThis` memo of the warmed cache never hits and was removed again.
  The cross-file mechanism Rails has is `db:schema:cache:dump` /
  `ActiveRecord.schema_cache_path` — a boot-produced dump each worker loads —
  which is filed as
  `0078-sti-schema-reflection-fidelity/warm-the-fixtures-schema-cache-from-a-boot-dump`
  and removes the cost for BOTH fixture paths, not just the one this story
  touches.

  Measured regression as shipped (sqlite, 2 runs each): `cache-key.test.ts`
  302/322ms -> 598/616ms, `migration.test.ts` 1120ms -> 1580/1402ms,
  `adapter.test.ts` 2939/2892ms -> 2753/3303ms. It applies to the ~78
  non-transactional fixture files only; every transactional file already paid
  the same `add_all`.
