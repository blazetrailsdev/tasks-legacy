---
title: "pg-add-foreign-key-delegate-to-abstract-body"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: null
priority: 50
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' PG `add_foreign_key` is two lines
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:578-582`):

```ruby
def add_foreign_key(from_table, to_table, **options)
  assert_valid_deferrable(options[:deferrable])
  super
end
```

trails' `packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts:1162-1204`
replicates the whole abstract body inline instead — the `use_foreign_keys?`
guard, the `if_not_exists` short-circuit, `foreignKeyOptions`,
`createAlterTable`, and `schemaCreation.accept(at)` are all duplicated.

The comment at :1187 justifies the duplication as avoiding recursion "through
the self-delegation guard", and cites
`abstract-add-foreign-key-converge-to-foreign-key-options` as the tracking
story. That story landed (#3803) and the guard is gone: abstract
`addForeignKey`
(`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:806-844`)
now says explicitly "there is no self-delegation here". The stated blocker no
longer exists, so the duplication is unjustified debt.

The one real obstacle left is the companion-class split: PG `SchemaStatements
extends AbstractSchemaStatements`, but the PG override reaches the adapter as
`this.pg.createAlterTable` / `this.pg.schemaCreation` while the abstract body
assumes `this` **is** the adapter (`this.createAlterTable`,
`this.schemaCreation`, `this.adapterName`). A bare `super.addForeignKey(...)`
therefore needs the `this` binding checked before it is safe — see
`project_mysql_schema_statements_overrides_unreachable_on_adapter` for the
same hazard on MySQL.

## Acceptance criteria

- PG `addForeignKey` reduces to Rails' shape: `assertValidDeferrable`, then a
  single delegation to the abstract body — no replicated guard,
  `foreignKeyOptions`, `createAlterTable`, or `schemaCreation.accept`.
- The `this`-binding split between the PG companion class and the abstract
  body is resolved (or the delegation routes through whatever binding the
  adapter actually exposes) so the abstract body runs against the PG adapter.
- The stale `abstract-add-foreign-key-converge-to-foreign-key-options` citation
  at :1187 is removed with the duplicated body.
- PG foreign-key tests stay green (`deferrable`, `NOT VALID`, action decoration,
  schema-qualified names, `if_not_exists`).
