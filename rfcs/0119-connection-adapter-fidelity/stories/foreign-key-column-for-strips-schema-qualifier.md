---
title: "foreign_key_column_for strips a schema qualifier Rails does not"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`foreignKeyColumnFor` (`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1555-1561`)
strips a leading schema qualifier before delegating:

```ts
const name = this.stripTableNamePrefixAndSuffix(tableName.replace(/^.*\./, ""));
```

Rails has no such strip
(`activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1241-1244`):

```ruby
def foreign_key_column_for(table_name, column_name) # :nodoc:
  name = strip_table_name_prefix_and_suffix(table_name)
  "#{name.singularize}_#{column_name}"
end
```

The `.replace(...)` is a trails addition (an inline comment in the file admits
it: "The leading schema-qualifier strip is a trails addition for PG's
`schema.table` form"). It surfaced as an RFC 0095 `naming` call-argument row —
Rails passes `table_name`, the port passes the result of `.replace` — and was
left in place by PR #6353 because it is an invented conversion, not a rename.

## Converged shape

```ts
const name = this.stripTableNamePrefixAndSuffix(tableName);
```

Before converging, establish whether any caller actually reaches this with a
`schema.table` argument. `new_foreign_key_definition` threads `to_table`
through, and PostgreSQL's own `foreign_keys` already returns bare identifiers,
so the strip may be dead. If a real PG path does depend on it, the fix belongs
at that call site (extracting the qualified name there, as
`postgresql/schema_statements.rb` does elsewhere), not inside a shared helper
Rails keeps qualifier-agnostic.

## Acceptance criteria

1. `foreignKeyColumnFor` passes `tableName` straight to
   `stripTableNamePrefixAndSuffix`, matching schema_statements.rb:1242.
2. Any PG path that genuinely needed the strip extracts the schema-qualified
   name at its own call site, with the Rails citation for that site.
3. The RFC 0095 `naming` row for `foreign_key_column_for` /
   `strip_table_name_prefix_and_suffix` is gone from
   `pnpm parity:api:calls:args:report`, not baselined.
