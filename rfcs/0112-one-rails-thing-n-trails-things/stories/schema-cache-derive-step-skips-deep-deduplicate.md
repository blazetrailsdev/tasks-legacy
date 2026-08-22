---
title: "SchemaCache derive step never calls deepDeduplicate"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 70
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `SchemaCache#derive_columns_hash_and_deduplicate_values`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/schema_cache.rb:440-446`)
runs `deep_deduplicate` over all four caches before exposing them:

```ruby
@columns      = deep_deduplicate(@columns)
@columns_hash = @columns.transform_values { |columns| columns.index_by(&:name) }
@primary_keys = deep_deduplicate(@primary_keys)
@data_sources = deep_deduplicate(@data_sources)
@indexes      = deep_deduplicate(@indexes)
```

trails' port
(`packages/activerecord/src/connection-adapters/schema-cache.ts`,
`deriveColumnsHashAndDeduplicateValues`) only rebuilds `_columnsHash` and
reconciles primary-key flags — it never calls the ported `deepDeduplicate`
helper (which exists lower in the same file, documented as the
`deep_deduplicate` port) on any of `_columns` / `_primaryKeys` /
`_dataSourceExists` / `_indexes`. The method name still claims it deduplicates.

`SchemaCache#indexes` (`:363-368` in Rails) also deep-deduplicates on the live
reflection path (`@indexes[deep_deduplicate(table_name)] =
deep_deduplicate(connection.indexes(table_name))`); trails' `indexes` stores the
adapter rows directly.

Noticed while porting index rehydration (PR #5890) — out of scope there because
the fix touches the column/primary-key/data-source caches too, not just
`_indexes`.

## Acceptance criteria

- `deriveColumnsHashAndDeduplicateValues` routes `_columns`, `_primaryKeys`,
  `_dataSourceExists` and `_indexes` through the ported `deepDeduplicate`, in
  Rails' order (columns first, then the hash derive).
- `SchemaCache#indexes` deep-deduplicates the reflected rows and the table-name
  key, mirroring `schema_cache.rb:367`.
- Deduplication must not defeat the `IndexDefinition` rehydration landed in
  #5890 — a deep-cloned index has to stay an `IndexDefinition` instance
  (`schema-cache.trails.test.ts` covers the round trip and must keep passing).
- A test asserts two tables caching structurally identical values share the
  deduplicated object.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
