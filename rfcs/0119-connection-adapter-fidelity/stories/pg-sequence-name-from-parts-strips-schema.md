---
title: "sequenceNameFromParts strips the schema qualifier; Rails uses table_name verbatim"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while fixing the hardcoded 63 in `sequenceNameFromParts` (RFC 0051
story `pg-sequence-name-from-parts-hardcodes-63`, PR #5475).

Rails' `sequence_name_from_parts`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:1007-1021`)
uses `table_name` verbatim — it never splits off a schema:

```ruby
def sequence_name_from_parts(table_name, column_name, suffix)
  over_length = [table_name, column_name, suffix].sum(&:length) + 2 - max_identifier_length
  ...
```

trails inserts an extra step
(`packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts`,
`sequenceNameFromParts`):

```ts
const [, unqualifiedTable] = this.pg.extractSchemaQualifiedName(tableName);
```

and then computes the budget and the returned name from `unqualifiedTable`. This
is a trails invention with two observable consequences on a schema-qualified
table name:

1. The overage math uses a shorter length than Rails does, so truncation kicks
   in at a different point.
2. The returned name drops the schema prefix, where Rails keeps whatever it was
   handed.

Both callers compare the result against a column's `nextval(...)` default to
decide whether a column is `serial:`
(`postgresql-adapter.ts:3981` via `sequenceNameFromParts`, and
`postgresql-adapter.ts:4840` in `newColumnFromField`), so the deviation can
change serial detection — and hence the dumped schema — for tables in a
non-default schema.

Investigate whether either call site depends on the stripping (it may have been
added to make a specific test pass); if it does, the fix belongs at the call
site, not inside the Rails-mirrored helper.

## Acceptance criteria

- `sequenceNameFromParts` uses `tableName` verbatim, matching
  `schema_statements.rb:1007-1021` line for line.
- Any call site that genuinely needs an unqualified name does the extraction
  itself, with the reason justified at the call site.
- A test pins the behaviour for a schema-qualified table name and fails on the
  stripping implementation.
