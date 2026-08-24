---
title: "sequence_name hard-codes the PostgreSQL convention instead of reset_sequence_name"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

# `sequence_name` hard-codes the PostgreSQL convention instead of `reset_sequence_name`

## Context

Rails' reader memoizes the adapter's answer:

```ruby
# activerecord/lib/active_record/model_schema.rb:371-382
def sequence_name
  if base_class?
    @sequence_name ||= reset_sequence_name
  else
    (@sequence_name ||= nil) || base_class.sequence_name
  end
end

def reset_sequence_name # :nodoc:
  @explicit_sequence_name = false
  @sequence_name          = with_connection { |c| c.default_sequence_name(table_name, primary_key) }
end
```

`default_sequence_name` is per-adapter: PostgreSQL derives it from
`pg_get_serial_sequence` / the `<table>_<pk>_seq` convention
(`postgresql/schema_statements.rb`), while the abstract and MySQL adapters
answer `nil` (`abstract/schema_statements.rb`).

trails' `sequenceName` (`packages/activerecord/src/model-schema.ts`) got Rails'
`base_class?` split in #6996, but the base-class arm still spells the
PostgreSQL convention inline — `` `${this.tableName}_${pk}_seq` `` — for every
adapter, and `reset_sequence_name` is never called from the reader. The reason
is a language shortcoming: `with_connection` is async in trails and the reader
is synchronous, so it cannot await `default_sequence_name`. That is stated at
the call site, and the call gate has no row for it only because the omission is
inside a body it does not flag.

Consequences: a MySQL/sqlite3 model reports a sequence name where Rails reports
`nil`, and a PostgreSQL model with a non-conventional sequence (an explicit
`nextval('other_seq')` default, a renamed sequence) reports the conventional
name rather than the one the database actually uses.

## Converged shape

- `resetSequenceName` calls the adapter's `defaultSequenceName(tableName, primaryKey)`
  and stores the result, exactly as `reset_sequence_name` does.
- `sequenceName`'s base-class arm becomes `_sequenceName ??= resetSequenceName()`,
  with no adapter convention spelled in `model-schema.ts`.
- Needs a synchronous path to the adapter — either the warm-cache read RFC 0073's
  permanent-checkout flip unlocks, or a `defaultSequenceName` warmed alongside
  the columns during `load_schema!` (the same trick `primary_key` uses through
  `SchemaCache#primaryKeys`).

## Acceptance criteria

- [ ] `model-schema.ts` contains no `_seq` string; the name comes from the adapter.
- [ ] `Model.sequenceName` is `null` on mysql2 and sqlite3 where the adapter's
      `default_sequence_name` answers nil, and the real sequence on postgresql.
- [ ] `base_test.rb`'s sequence-name family ("sequence name", "clear cache when
      setting table name", "dont clear sequence name when setting explicitly")
      passes on all three lanes.
