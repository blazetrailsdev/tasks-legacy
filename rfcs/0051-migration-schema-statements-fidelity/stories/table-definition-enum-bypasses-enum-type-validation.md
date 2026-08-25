---
title: "TableDefinition#enum substitutes the enum name for the column type, bypassing enum_type validation"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 60
priority: 15
pr: 7018
claim: "2026-08-25T00:06:07Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-naming-test-file"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by review of PR #5624 (`pg-column-methods-on-change-table-proxy`, RFC 0005).

That PR fixed `PostgreSQL::Table#enum` to stay on Rails' generated
`define_column_methods` path — `column(name, :enum, **options)` unchanged, letting
PG `type_to_sql` validate and resolve `enum_type`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:854-857`).

The **same defect is still present on the `create_table` path**. Abstract
`TableDefinition#enum`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:1465-1468`)
does:

```ts
enum(name: string, options: ColumnOptions & { enum_type: string }): this {
  const { enum_type: enumType, ...rest } = options;
  return this.column(name, enumType as ColumnType, rest);
}
```

It substitutes the enum's name for the column type up front, so:

- the `enum_type is required for enums` validation is unreachable — a missing
  `enum_type` silently produces a column whose type is `undefined` rather than raising;
- the column's `type` is the enum name instead of `enum`, unlike Rails.

Rails has no `enum` override on `TableDefinition` at all — `:enum` is one of the
names in PG's `define_column_methods` list
(`postgresql/schema_definitions.rb:186-191`), so the generated method applies on
both `TableDefinition` and `Table`.

Note this method is on the **abstract** TableDefinition, which is itself
suspicious: `enum` is PG-only in Rails.

## Acceptance criteria

- `t.enum` on the `create_table` path passes `:enum` through as the column type with
  `enum_type` forwarded, so PG `type_to_sql` performs the validation and resolution
  (mirroring the `Table#enum` shape landed in #5624).
- A missing `enum_type` raises `ArgumentError("enum_type is required for enums")`
  rather than producing a column of another type.
- Consider relocating `enum` from abstract `TableDefinition` to PG's
  `TableDefinition`, where Rails puts it.
- Schema-dump round-trip still works: the PG dumper emits `enum_type:`
  (`postgresql/schema-dumper.ts:31,38`) and that output must keep loading.
- `parity:api` / `parity:test` deltas non-negative.
