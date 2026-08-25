---
title: "get_primary_key drops Rails' table_exists? reflection branch"
status: draft
updated: 2026-07-29
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

# get_primary_key drops Rails' table_exists? reflection branch

## Context

Surfaced by PR #5568 (`port-migration-change-schema-type-reflection-cases`),
which wired `getPrimaryKey` onto `Base` and into `TableDefinition` so
`Base.primaryKeyPrefixType` reaches `create_table` DDL.

Rails' `get_primary_key`
(`vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb`)
has four branches:

```ruby
def get_primary_key(base_name)
  if base_name && primary_key_prefix_type == :table_name
    base_name.foreign_key(false)
  elsif base_name && primary_key_prefix_type == :table_name_with_underscore
    base_name.foreign_key
  elsif ActiveRecord::Base != self && table_exists?
    pk = connection_pool.schema_cache.primary_keys(table_name)
    suppress_composite_primary_key(pk)
  else
    "id"
  end
end
```

trails' port
(`packages/activerecord/src/attribute-methods/primary-key.ts`, `getPrimaryKey`)
implements only the two prefix branches and the `"id"` fallback — the third
branch, which reflects the real PK off an existing table via the schema cache,
is missing entirely.

It is unreachable from the call site PR #5568 added: `TableDefinition` calls it
as `Base.getPrimaryKey(...)`, so `ActiveRecord::Base != self` is false, and
`set_primary_key` runs while the table is being created, so `table_exists?`
would be false anyway. But `Base.getPrimaryKey` is now public class-method
surface, and a **subclass** receiver (`Post.getPrimaryKey("post")` against an
existing table with a non-`id` PK) silently returns `"id"` where Rails returns
the reflected key.

The blocker is shape, not logic: `table_exists?` and
`schema_cache.primary_keys` are async in trails, and `getPrimaryKey` is
synchronous because `TableDefinition`'s constructor calls it synchronously.
Closing this needs a decision about which side becomes async (or whether the
branch can be served from an already-warmed schema cache synchronously).

## Acceptance criteria

- [ ] `getPrimaryKey` implements the `ActiveRecord::Base != self && table_exists?`
      branch, including `suppress_composite_primary_key`.
- [ ] The sync/async shape conflict with `TableDefinition`'s constructor is
      resolved without making the constructor async, or the deviation is
      documented at the call site with the reason it cannot be.
- [ ] A test covers a subclass receiver whose table has a non-`id` primary key,
      and fails on baseline.
- [ ] Green on all three lanes.
