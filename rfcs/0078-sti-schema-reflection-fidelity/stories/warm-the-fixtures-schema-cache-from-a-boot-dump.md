---
title: "warm-the-fixtures-schema-cache-from-a-boot-dump"
status: done
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6972
claim: "2026-08-24T03:39:39Z"
assignee: "warm-the-fixtures-schema-cache-from-a-boot-dump"
blocked-by: null
closed-reason: null
---

## Context

PR #6936 (`tablename-only-model-resolves-columns-without-explicit-reflect`,
RFC 0078) extended the eager `SchemaCache#add_all` warm to the
non-transactional fixtures path, because the sync `load_schema`
(`packages/activerecord/src/model-schema.ts` `loadSchemaFromCacheSync`) can
only answer from that cache — a model whose only class-level declaration is
`tableName` reflects no columns at all when it is cold.

The warm costs ~0.3s per test file: `SchemaCache#add` issues four
introspection queries (`dataSourceExists`, `primaryKeys`, `columns`,
`indexes`) per table, over the ~200 canonical tables. Measured on sqlite,
2 runs each:

- `cache-key.test.ts` 302/322ms -> 598/616ms
- `migration.test.ts` 1120ms -> 1580/1402ms

Every transactional fixtures file has always paid this; #6936 added ~78
non-transactional files to the set. The story's "no measurable per-test-file
cost regression" criterion is therefore not met, and the cost is worth
removing for BOTH paths, not just re-narrowing.

Rails' answer is `db:schema:cache:dump` / `ActiveRecord.schema_cache_path`:
the cache is dumped once and loaded per process. trails already has both
halves — `SchemaCache#dumpTo` / `SchemaCache._loadFrom`
(`packages/activerecord/src/connection-adapters/schema-cache.ts`) — and the
canonical schema is laid into a template DB once at boot and cloned into
every worker slot, so one dump is valid for every worker of that adapter.

Note vitest's `isolate: true` reloads the module graph per file but reuses
the worker PROCESS, so `globalThis` persists across files
(`packages/activerecord/src/support/sqlite-template.ts:190-192` relies on
this) — an in-process memo is a cheaper variant to consider, with the
caveat that a file which ran DDL would poison later files' caches.

## Acceptance criteria

- The per-file eager warm no longer issues ~800 introspection queries: it
  loads a boot-produced cache (or an equivalent per-worker memo) instead.
- Files that lay their own DDL still see a correct schema — a table created
  or altered inside a test file must not be answered from a stale entry.
- State the numbers: the files above are back at or below their pre-#6936
  timings, and transactional files improve too.
