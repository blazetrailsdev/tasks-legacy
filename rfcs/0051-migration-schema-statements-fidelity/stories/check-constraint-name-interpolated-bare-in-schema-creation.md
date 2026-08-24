---
title: "visit_CheckConstraintDefinition interpolates the constraint name bare, not quoted"
status: claimed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 40
priority: 13
pr: null
claim: "2026-08-24T23:42:13Z"
assignee: "move-ts-only-extras-out-of-the-remaining-activemodel-type-test-files-part-2"
blocked-by: null
closed-reason: null
---

## Context

`SchemaCreation#visitCheckConstraintDefinition`
(`packages/activerecord/src/connection-adapters/abstract/schema-creation.ts`)
emits

```ts
return `CONSTRAINT ${this.adapter.quoteColumnName(o.name)} CHECK (${o.expression})`;
```

Rails' `visit_CheckConstraintDefinition`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_creation.rb:125-127`)
interpolates the name bare:

```ruby
def visit_CheckConstraintDefinition(o)
  "CONSTRAINT #{o.name} CHECK (#{o.expression})"
end
```

Neither PostgreSQL, MySQL, nor SQLite overrides it. Surfaced while porting
`migration/check_constraint_test.rb` (PR #6145), which removed the _other_
invented clause in this same method — a `validate: false` guard that raised on
non-PostgreSQL adapters and has no Rails counterpart. The quoting was left in
place because it is out of that PR's scope and changes emitted DDL on every
adapter.

## Converged shape

`visitCheckConstraintDefinition` interpolates `o.name` bare, per
schema_creation.rb:125-127.

## Acceptance criteria

- [ ] The `quoteColumnName` call is gone; the name is interpolated as Rails does.
- [ ] `migration/check-constraint.test.ts`, `adapters/mysql2/check-constraint-quoting.test.ts`,
      and the schema-dumper suites stay green on all three lanes — check whether
      any of them assert the quoted form and, if so, whether the assertion has a
      Rails counterpart before changing it.
