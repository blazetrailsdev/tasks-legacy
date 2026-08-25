---
title: "PG TableDefinition takes unlogged as a ctor option; Rails reads PostgreSQLAdapter.create_unlogged_tables directly"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7042
claim: "2026-08-25T15:38:33Z"
assignee: "pg-table-definition-takes-unlogged-as-an-option-rails-reads-the-adapter"
blocked-by: null
closed-reason: null
---

## Context

Rails' `PostgreSQL::TableDefinition#initialize` reads the unlogged setting off
the adapter class itself, inside the constructor
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:250-255`):

```ruby
def initialize(*, **)
  super
  @exclusion_constraints = []
  @unique_constraints = []
  @unlogged = ActiveRecord::ConnectionAdapters::PostgreSQLAdapter.create_unlogged_tables
end
```

There is no `unlogged:` kwarg — the value is not caller-supplied at all, and
`PostgreSQL::SchemaStatements#create_table_definition` passes nothing for it.

trails instead threads it in as an option from the adapter
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:3703`):

```ts
createTableDefinition(name: string, options: Record<string, unknown> = {}): PgTableDefinition {
  const rest = options;
  const unlogged =
    (rest.unlogged as boolean | undefined) ?? PostgreSQLAdapter.createUnloggedTables;
  return new PgTableDefinition(this, name, { ...rest, unlogged });
}
```

and the ctor reads it back out
(`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:239-241`):

```ts
unlogged?: boolean;   // in the options bag
...
this.unlogged = options.unlogged ?? false;
```

Two consequences: `unlogged` is a caller-overridable option in trails where
Rails has no such seam at all, and the `?? false` default silently disagrees
with Rails whenever the ctor is reached without the adapter having injected the
value (every direct `new PgTableDefinition(conn, name)` in a test).

Surfaced while landing `table-definition-takes-conn-positionally-and-invents-adaptername`
(PR #7033), which moved the ctor to Rails' positional shape; the `unlogged`
option survived that move untouched and is the remaining divergence in the
signature.

## Converged shape

`PostgreSQL::TableDefinition`'s constructor reads
`PostgreSQLAdapter.createUnloggedTables` directly, as Rails does, and drops the
`unlogged` option from both the ctor's options type and the
`createTableDefinition` override — which then becomes the plain
`new PgTableDefinition(this, name, options)` that mirrors
`postgresql/schema_statements.rb`'s `super`.

Watch for the module cycle: `schema-definitions.ts` reading
`PostgreSQLAdapter` is an import edge back into `postgresql-adapter.ts`. If it
closes a cycle, the settled shape is the zero-import slot module (CLAUDE.md,
"Call-time constant resolution"), not a re-introduced option.

## Acceptance criteria

- [ ] `PostgreSQL::TableDefinition`'s ctor takes no `unlogged` option; the field
      is initialised from `PostgreSQLAdapter.createUnloggedTables`.
- [ ] `PostgreSQLAdapter#createTableDefinition` no longer computes or forwards
      `unlogged`.
- [ ] A direct `new PgTableDefinition(conn, name)` reflects the adapter's
      current `createUnloggedTables`, not `false`.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
