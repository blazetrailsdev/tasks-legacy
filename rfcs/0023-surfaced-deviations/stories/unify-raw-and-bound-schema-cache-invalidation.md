---
title: "Unify raw/bound schema-cache invalidation so uniqueness reads schemaCacheBound"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while retiring `poolAbsent` / `realPool` in #5885.

`tableIndexes` in `packages/activerecord/src/validations/uniqueness.ts` reads
the _raw_ `SchemaCache` rather than the Rails-shaped one-arg
`adapter.schemaCacheBound`, and its JSDoc records why:

> `addIndex` invalidates only the raw cache
> (`adapter.schemaCache.clearDataSourceCacheBang`), so the bound reflection can
> serve a stale, pre-index list and silently keep the optimization off after a
> migration adds the covering index.

Rails has no such split. `SchemaCache#clear_data_source_cache!`
(`schema_cache.rb:218`) is reached through `BoundSchemaReflection`, which
delegates to the same `SchemaReflection` every reader goes through
(`schema_cache.rb:160-222`), so an invalidation is visible to every reader by
construction. trails has two invalidation surfaces that can drift apart, and
the uniqueness validator is currently routed around the problem rather than the
problem being fixed.

This matters beyond style: Rails' `covered_by_unique_index?`
(`validations/uniqueness.rb:70-90`) is a correctness-relevant optimization —
if it reads a stale index list it silently changes which query the validator
issues.

## Acceptance criteria

- Establish, against `schema_cache.rb:160-222`, whether trails' raw-cache and
  bound-reflection invalidation paths can be unified so a `clearDataSourceCacheBang`
  through either is visible to both.
- If unifiable: unify them, and move `tableIndexes` onto `schemaCacheBound`
  (Rails' `klass.schema_cache.indexes(klass.table_name)`), dropping the
  raw-cache workaround and its JSDoc rationale.
- Add a regression test that fails on the current baseline: add a covering
  unique index after the cache is warm, then assert the uniqueness validator
  sees it.
- If not unifiable, record the real reason at `tableIndexes`.
- No test renames.
