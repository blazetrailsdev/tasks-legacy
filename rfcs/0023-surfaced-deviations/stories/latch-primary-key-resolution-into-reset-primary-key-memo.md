---
title: "primary_key re-resolves on every read instead of latching into @primary_key"
status: draft
updated: 2026-08-21
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

Surfaced by PR #6842 (`wave-4c-ar-core-residue-attributes-remainder-part-5`).

Rails resolves `primary_key` ONCE and latches the answer into `@primary_key`,
`vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb:77-98`:

```ruby
def primary_key
  reset_primary_key if PRIMARY_KEY_NOT_SET.equal?(@primary_key)
  @primary_key
end

def reset_primary_key # :nodoc:
  if base_class?
    self.primary_key = get_primary_key(base_class.name)
  else
    self.primary_key = base_class.primary_key
  end
end
```

`PRIMARY_KEY_NOT_SET` is a `BasicObject` sentinel, so a resolved `nil` (a
key-less view) is latched too and never re-resolved.

trails' `getPrimaryKeyAttr`
(`packages/activerecord/src/attribute-methods/primary-key.ts`) instead
re-resolves through `getPrimaryKey` on EVERY read and never writes
`_primaryKey`:

```ts
const configured = this._primaryKey;
if (configured !== undefined) return configured;
return getPrimaryKey.call(this, baseClass.call(this).name);
```

The reason is stated in its JSDoc and is real: `table_exists?` is async in
trails, so the schema cache can be cold at the time of the first
`primary_key` read (model construction runs before any awaited schema load).
Latching there would cache the `"id"` convention forever for a model whose
real key is `nick` or whose key is `nil`. `PrimaryKeysTest > primary key
returns nil if it does not exist` already has to `await` a schema-cache warm
by hand to compensate.

Consequences of the deviation:

- `primary_key` is not a memo, so every read walks `baseClass()` and the
  schema cache. It is on hot paths (`_read_attribute(@primary_key)`).
- `undefined` is the "not set" sentinel, so it cannot distinguish
  "unresolved" from "resolved to nil" the way Rails' `PRIMARY_KEY_NOT_SET`
  `BasicObject` does. A resolved `null` is re-resolved on each read.

Converging most likely depends on the schema cache being warm before the
first `primary_key` read — see the sibling story on `get_primary_key`'s
lease-free schema-cache arm, which this one is coupled to.

## Acceptance criteria

- [ ] `primary_key` latches its resolved value the way primary_key.rb:77-80
      does, with a `PRIMARY_KEY_NOT_SET`-equivalent sentinel that survives a
      resolved `null`, or the remaining gap is a demonstrated language
      shortcoming re-justified at the call site.
- [ ] `getPrimaryKeyAttr`'s "resolves on EVERY read" JSDoc note is removed or
      replaced by the converged rationale.
- [ ] `PrimaryKeysTest > primary key returns nil if it does not exist` no
      longer needs its hand-rolled schema-cache warm, or that warm is
      justified against the Rails test.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
