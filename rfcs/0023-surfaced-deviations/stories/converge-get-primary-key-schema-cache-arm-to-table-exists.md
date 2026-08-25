---
title: "get_primary_key's schema-cache arm substitutes a lease-free cache read for table_exists?/primary_keys"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6842 (`wave-4c-ar-core-residue-attributes-remainder-part-5`),
which converged the eight `kind: "set"` call rows in
`call-mismatches-exclude/activerecord/attribute-methods/primary-key.json` and
gave `getPrimaryKey` Rails' four-branch shape. Two calls in the third branch
did not converge and ship as `@missingRailsCall` tags at the call site in
`packages/activerecord/src/attribute-methods/primary-key.ts`.

Rails, `vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb:101-108`:

```ruby
def get_primary_key(base_name) # :nodoc:
  if base_name && primary_key_prefix_type == :table_name
    base_name.foreign_key(false)
  elsif base_name && primary_key_prefix_type == :table_name_with_underscore
    base_name.foreign_key
  elsif ActiveRecord::Base != self && table_exists?
    schema_cache.primary_keys(table_name)
  else
    "id"
  end
end
```

trails ships the third branch as a lease-free, cache-only read:

```ts
const primaryKeys = cachedSchemaCacheFor(this)?.getCachedPrimaryKeys?.(tableName);
if (primaryKeys !== undefined) return primaryKeys;
```

Two forces produced that shape, and both are real:

1. `table_exists?` and `schema_cache.primary_keys` are both async in trails,
   while `primary_key` is read from synchronous hot paths (model construction,
   `_read_attribute(@primary_key)`).
2. `cachedTableExists` (`model-schema.ts`), the sync cache-only view of
   `table_exists?`, resolves its adapter through `reflectionAdapter`, which
   falls through to `pool.leaseConnectionSync()`. Routing `primary_key`
   through it makes every primary-key read permanently check out a connection
   — PR #6842 shipped exactly that in its first push and reddened
   `ConnectionHandlingTest > common APIs don't permanently hold a connection
when permanent checkout is deprecated or disallowed` on all three lanes
   before it was backed out. See the `cachedSchemaCacheFor` docblock in
   `primary-key.ts`, which warns about this.

So the convergence needs a lease-free sync path to `table_exists?` — not just
an async one. Note the older draft story
`get-primary-key-missing-table-exists-branch` (0023) predates #6842 and is
mostly landed by it; triage should narrow or retire it in favour of this
residue-only scope.

## Acceptance criteria

- [ ] `getPrimaryKey`'s third branch reads `table_exists?` and
      `schema_cache.primary_keys(table_name)` under their Rails names, or the
      remaining gap is a demonstrated language shortcoming re-justified at the
      call site.
- [ ] Both `@missingRailsCall` tags (`table_exists?`, `primary_keys`) are
      removed from `getPrimaryKey` in
      `packages/activerecord/src/attribute-methods/primary-key.ts`.
- [ ] `ConnectionHandlingTest > common APIs don't permanently hold a connection
when permanent checkout is deprecated or disallowed` stays green — no
      `leaseConnectionSync` on the `primary_key` read path.
- [ ] `pnpm parity:api:calls` green with no new baseline row.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
