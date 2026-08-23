---
title: "tablename-only-model-resolves-columns-without-explicit-reflect"
status: in-progress
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
- No measurable per-test-file cost regression from warming (state the numbers).
