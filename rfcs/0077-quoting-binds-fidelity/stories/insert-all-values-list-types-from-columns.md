---
title: "values_list resolves types from columns, not attribute definitions"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 6
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`InsertAll::Builder#values_list` resolves each key's type from the model's
attribute definitions:

```ts
const def = model._attributeDefinitions.get(key) as any;
const type = def?.type ?? def;
```

Rails resolves it from the **table's columns**, via a dedicated helper:

```ruby
def values_list
  types = extract_types_from_columns_on(model.table_name, keys: keys_including_timestamps)

  values_list = insert_all.map_key_with_value do |key, value|
    next value if Arel::Nodes::SqlLiteral === value
    ActiveModel::Type::SerializeCastValue.serialize(type = types[key], type.cast(value))
  end

  connection.visitor.compile(Arel::Nodes::ValuesList.new(values_list))
end
```

(`activerecord/lib/active_record/insert_all.rb:238-247`; the helper is
`extract_types_from_columns_on`, `insert_all.rb:281-287`.)

The two disagree whenever a column exists in the table but has no declared
attribute definition — Rails still finds a type through the column, trails
falls through to the untyped path. It also disagrees for an attribute
declared with a type that differs from the column's.

Surfaced by #6294, which converged the _quoting_ half of `values_list` (the
values now go to the visitor unquoted, per rb:246) but left type resolution
untouched.

## Converged shape

Port `extract_types_from_columns_on(table_name, keys:)` (insert_all.rb:281-287)
and use its `types[key]` in `values_list`, replacing the
`_attributeDefinitions` lookup.

Rails then calls `SerializeCastValue.serialize(type, type.cast(value))`
unconditionally — no nil-type arm, because `extract_types_from_columns_on`
always yields a type. trails' three-branch fallback (`type.serialize` /
`type.serializeCastValue` / identity `SerializeCastValue.serializeCastValue`)
exists only to tolerate the missing type this change removes, so it collapses
to the single rb:241 call.

## Acceptance criteria

- [ ] `extract_types_from_columns_on` is ported at the Rails name and used by
      `values_list`; `_attributeDefinitions` is no longer consulted there.
- [ ] The serialize call is the unconditional rb:241
      `SerializeCastValue.serialize(type, type.cast(value))`.
- [ ] `insert_all` / `upsert_all` green on all three adapters, including a
      column present in the table but not declared as an attribute.
- [ ] parity:api / parity:test delta non-negative.
