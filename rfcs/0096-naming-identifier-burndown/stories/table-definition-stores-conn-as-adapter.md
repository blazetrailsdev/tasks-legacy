---
title: "TableDefinition and SchemaCreation store Rails' @conn under a trails name (_adapter / adapter)"
status: done
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: 7040
claim: "2026-08-25T15:35:31Z"
assignee: "table-definition-stores-conn-as-adapter"
blocked-by: null
closed-reason: null
---

## Context

Rails' `TableDefinition` stores the connection as `@conn` and reads it under
that name throughout
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:371-393`):

```ruby
def initialize(conn, name, ...)
  @conn = conn
  ...
end
```

with every body reading `@conn` — e.g. `@conn.valid_column_definition_options`,
`@conn.supports_datetime_with_precision?`.

trails stores the same value as `_adapter`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:1089`,
assigned at :1105) and reads it as `this._adapter` at 6 sites in
`abstract/schema-definitions.ts` (:1237, :1302, and 4 more) plus 2 in
`postgresql/schema-definitions.ts`. `SchemaCreation` has the same split — it
reads `this.adapter` where Rails reads `@conn`
(`abstract/schema_creation.rb:8-18`).

`table-definition-takes-conn-positionally-and-invents-adaptername` (PR #7033)
converged the constructor's parameter to Rails' `conn` and made it positional,
but stopped at the parameter: the field it is assigned to still carries the
trails name, so the ctor now reads `this._adapter = conn` — the rename is
visible in one line and undone in the next.

This is the "locals and parameters keep the Rails identifier" rule in CLAUDE.md
applied to the ivar, and it is free fidelity: the field is `protected`, so no
public surface moves and `parity:api` cannot see the change either way.

## Converged shape

`TableDefinition`'s connection field is named `conn`, matching `@conn`, with
all 8 read sites renamed. `SchemaCreation`'s `adapter` field likewise becomes
`conn` to match `abstract/schema_creation.rb`. No behavior change, no signature
change — a pure rename, mechanical enough to note as such in the PR body.

Check `TableDefinitionConn` (the host interface type) at the same time: the
type name is trails-only scaffolding for a Ruby duck type and may want the same
treatment or none at all.

## Acceptance criteria

- [ ] `TableDefinition`'s connection field is `conn`; no `_adapter` remains in
      `abstract/`, `postgresql/`, `mysql/` or `sqlite3/schema-definitions.ts`.
- [ ] `SchemaCreation`'s field is `conn`, matching `schema_creation.rb`.
- [ ] `pnpm parity:api` activerecord matched count does not decrease;
      `parity:api:calls` / `:calls:args` non-negative.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
