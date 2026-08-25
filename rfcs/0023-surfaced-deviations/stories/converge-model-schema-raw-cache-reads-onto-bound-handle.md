---
title: "Converge model-schema's async raw-cache reads onto the bound handle"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Left over from PR #5906 (story
`converge-schema-cache-getter-onto-bound-reflection`). That PR converged
`AbstractAdapter#schemaCache` onto the bound reflection and moved the adapter
DDL sites, `tableExists`, `insertAll`, the uniqueness validator and
`batches.rb:315` onto Rails' one-arg forms.

`packages/activerecord/src/model-schema.ts` still holds a cluster of raw-cache
readers on the trails-only `internalSchemaCache` name — roughly ten call sites,
including the `clearDataSourceCacheBang(pool, name)` bust inside the
`createTable` helper and the `columnsHash(pool, table)` read in the virtual-
attribute reconciliation path.

Two distinct reasons they were left behind, and they need splitting:

1. Sync peeks (`getCachedColumnsHash`, `getCachedDataSourceExists`) have no
   `BoundSchemaReflection` equivalent — every bound read is async. These are
   the RFC 0023 sync/async deviation and likely belong with
   `retire-schema-cache-sync-and-ledger-shims`.
2. Genuinely async readers that just weren't converted (the `columnsHash` read
   and the `clearDataSourceCacheBang` bust). These are mechanical: Rails uses
   `schema_cache.columns_hash(table_name)` /
   `schema_cache.clear_data_source_cache!(table_name)`.

## Acceptance criteria

- Every async raw-cache reader in `model-schema.ts` moves to
  `schemaCache` (the bound handle, one-arg form).
- Each remaining `internalSchemaCache` reader is a sync peek, and carries a
  one-line note at the call site saying so.
- No behavior change to the shared-slot invariant PR #5906 established:
  `PoolConfig#schemaCache`, `internalSchemaCache` and the bound handle must
  keep reading one `SchemaReflection` cache, so a DDL bust stays visible to
  every reader. `UniquenessValidationWithIndexTest` is the regression canary.
