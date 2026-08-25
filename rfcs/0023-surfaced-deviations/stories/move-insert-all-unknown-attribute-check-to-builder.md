---
title: "Move insert_all's UnknownAttributeError to Rails' Builder-time reflected-columns check"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Move `insert_all`'s UnknownAttributeError to Rails' Builder-time reflected-columns check

## Context

Rails raises `UnknownAttributeError` in `InsertAll::Builder`, not in
`InsertAll#initialize`:

```ruby
# activerecord/lib/active_record/insert_all.rb:239-244
def types_for_columns_on(table_name, keys)
  columns = extract_types_from_columns_on(table_name, keys: keys)
  ...
end

# activerecord/lib/active_record/insert_all.rb:306-313
def extract_types_from_columns_on(table_name, keys:)
  columns = @model.schema_cache.columns_hash(table_name)

  unknown_column = (keys - columns.keys).first
  raise UnknownAttributeError.new(model.new, unknown_column) if unknown_column

  keys.index_with { |key| model.type_for_attribute(key) }
end
```

`schema_cache.columns_hash` is a **blocking** read: it reflects the table if the
cache is cold, so Rails always judges the keys against real reflected columns.
`initialize` (insert_all.rb:18-46) contains no such check.

trails instead checks in the constructor
(`packages/activerecord/src/insert-all.ts` `verifyAttributeNamesAreKnown`),
against `model.attributeNames()` — a synchronous reader that answers only from
an already-loaded schema. #6996 had to add an `isSchemaLoaded` early return
there, because `table_name=` running Rails' `reset_column_information`
(model_schema.rb:273-281) leaves the model unloaded until the next async
reflect, and the constructor was raising on a column that reflect was about to
produce (`insert_all_test.rb`'s "insert all when table name contains database",
mysql-only).

The residue: an **unloaded** model given a genuinely unknown key now bypasses
the guard entirely, and `Builder.valuesList()` only calls
`model.typeForAttribute(key)` before generating SQL — so Rails'
`UnknownAttributeError` is either missed or degraded into an adapter SQL error.

## Converged shape

- Delete `verifyAttributeNamesAreKnown` and its constructor call site.
- Raise where Rails raises: in the Builder's types-for-columns path, after the
  awaited reflect that `execute()` already performs
  (`_populateUpdatableColumns`), against `columnsHash` **keys** — Rails' set —
  rather than `attributeNames()`.
- Message and record shape stay as they are (`UnknownAttributeError.new(model.new, unknown_column)`).

Watch the throw site moving from constructor to `await`: `insertAll` /
`insertAllBang` / `upsertAll` already return promises, so callers see a
rejection either way, but any assertion expecting a synchronous throw needs
re-checking.

## Acceptance criteria

- [ ] No unknown-attribute check in `InsertAll`'s constructor.
- [ ] An unknown key raises `UnknownAttributeError` for a model whose schema was
      NOT loaded at call time (the case #6996's `isSchemaLoaded` guard skips) —
      regression test must fail on that baseline.
- [ ] `insert-all.test.ts` and `insert-all.trails.test.ts` pass on sqlite3,
      postgresql and mysql2, including "insert all when table name contains
      database".
