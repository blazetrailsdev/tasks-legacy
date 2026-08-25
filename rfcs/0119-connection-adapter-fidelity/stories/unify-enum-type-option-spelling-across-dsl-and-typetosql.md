---
title: "enum_type option is renamed to enumType only at the typeToSql hop, forcing every DSL caller to rekey"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Surfaced by review of PR #5624 (`pg-column-methods-on-change-table-proxy`, RFC 0005).

Rails names this option `enum_type:` consistently: the DSL takes
`t.enum :best_color, enum_type: "color"`, `type_to_sql` receives it as the
`enum_type:` kwarg
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:830,854-857`),
and the schema dumper emits `enum_type:`.

trails renamed only the middle hop: `PostgreSQLSchemaStatements#typeToSql` takes
`enumType` (`packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts:825-834,862`),
while the DSL-facing option and the dumper output stay `enum_type`
(`postgresql/schema-dumper.ts:31,38`; `postgresql/schema-definitions.ts:97`).
So every DSL entry point has to rekey across the boundary — `Table#enum`
(schema-definitions.ts) now does `{ ...rest, enumType }`, and abstract
`TableDefinition#enum` destructures `enum_type` into a local `enumType`.

This is a naming seam with no Rails counterpart. Each new caller has to rediscover
it, and forgetting the rekey fails _silently at the type level_ (the extra key is
just ignored by `ColumnOptions`, which has no index signature) and then blows up
deep inside `SchemaCreation` with `enum_type is required for enums`.

## Acceptance criteria

- One spelling of the option across the DSL → `typeToSql` → dumper path, so no
  call site rekeys. Prefer whichever keeps the dumper output matching Rails'
  `schema.rb` (`enum_type:`) while satisfying the repo's camelCase rule for
  internal kwargs — state the choice at the call site.
- No DSL entry point silently drops the option; a missing value still raises
  `ArgumentError("enum_type is required for enums")`.
- `parity:api` / `parity:test` deltas non-negative.
