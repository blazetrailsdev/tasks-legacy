---
title: "InsertAll::Builder never ports extract_types_from_columns_on — no types memo, and the unknown-column raise sits elsewhere"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 130
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6769, which collapsed `InsertAll::Builder#valuesList` onto Rails'
single serialize line:

```ts
const type = model.typeForAttribute(key);
value = SerializeCastValue.serialize(type, type.cast(value));
```

That matches `insert_all.rb:242-243` for the per-value work, but Rails resolves the
type map ONCE, up front, through a private helper trails has never ported
(`vendor/rails/activerecord/lib/active_record/insert_all.rb:305-313`):

```ruby
def extract_types_from_columns_on(table_name, keys:)
  columns = @model.schema_cache.columns_hash(table_name)

  unknown_column = (keys - columns.keys).first
  raise UnknownAttributeError.new(model.new, unknown_column) if unknown_column

  keys.index_with { |key| model.type_for_attribute(key) }
end
```

`values_list` then reads `types[key]` (`insert_all.rb:242`) out of that memo.

Two consequences of the absence:

1. **No `types` memo.** trails calls `model.typeForAttribute(key)` once per key per
   ROW, where Rails builds the map once for the whole statement. On a large
   `insert_all` this is N_rows x N_keys `attribute_types` proxy reads.
2. **The unknown-column raise lives elsewhere.** Rails raises `UnknownAttributeError`
   from `extract_types_from_columns_on`, keyed on the SCHEMA CACHE's
   `columns_hash(table_name)`. trails raises the same error earlier, from the
   pre-existing `verifyAttributeNamesAreKnown` (`insert-all.ts:288-304`). Reviewed on
   #6769 and confirmed not a behavioral regression for the covered cases, but it is a
   different raise site keyed on a different source, so the two can disagree — an
   attribute the model declares but the table lacks, or vice versa.

## Converged shape

Port `extract_types_from_columns_on(table_name, keys:)` with the Rails name and
signature, reading `model.schemaCache.columnsHash(tableName)`; have `valuesList`
read `types[key]` from it. Then decide whether `verifyAttributeNamesAreKnown` is
still needed or whether the Rails raise site subsumes it — if it stays, justify it at
the call site, and if it goes, confirm the schema-cache-keyed raise covers every case
it did.

## Acceptance criteria

- [ ] `extractTypesFromColumnsOn` exists with Rails' name, params and raise, and
      `valuesList` reads `types[key]` from its result.
- [ ] The type map is built once per statement, not per row.
- [ ] `UnknownAttributeError` is raised for an unknown column with Rails' message,
      keyed on the schema cache's `columns_hash`, covered by a test.
- [ ] `verifyAttributeNamesAreKnown` is either deleted or justified at its call site
      with a Rails cite.
- [ ] `pnpm parity:api:calls` / `:args` green; `pnpm parity:api:extra --package
activerecord` shows no new surface; SQLite, PostgreSQL and MySQL lanes green.
