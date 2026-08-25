---
title: "Abstract rename_table ships a working body where Rails raises NotImplementedError"
status: claimed
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: 32
pr: null
claim: "2026-08-25T00:54:07Z"
assignee: "split-model-mixin-surface-to-active-model-model"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while reviewing PR #5574 (`port-migration-rename-table-cases`).

Rails' abstract `SchemaStatements#rename_table` is a pure abstract hook:

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:524-526`

```ruby
def rename_table(table_name, new_name, **)
  raise NotImplementedError, "rename_table is not implemented"
end
```

Every concrete adapter overrides it (sqlite3_adapter.rb:330,
abstract_mysql_adapter.rb:331, postgresql/schema_statements.rb:434).

trails instead ships a _working_ generic body at
`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:592`:
it clears the two schema-cache entries and emits
`ALTER TABLE <old> RENAME TO <new>`. That is a trails invention — Rails never
runs a generic rename — and it silently diverges from the adapter overrides,
which additionally call `validate_table_length!` and `rename_table_indexes`
(both wired up by PR #5574). Any caller reaching the base implementation
therefore gets an unvalidated rename that leaves auto-named indexes pointing at
the old table name.

The risk is that the base body masks a missing override rather than failing
loudly the way Rails does.

## Acceptance criteria

- [ ] Determine whether anything actually reaches the base `renameTable`
      (all three adapters override it; check the `AbstractAdapter` interface at
      `connection-adapters/abstract-adapter.ts:237` and any test doubles).
- [ ] If nothing depends on the generic body, replace it with the Rails
      `NotImplementedError` throw.
- [ ] If callers do depend on it, document why at the call site per the
      deviation convention rather than leaving it silent.
- [ ] Green on all three lanes; `parity:api` / `parity:test` delta
      non-negative.
