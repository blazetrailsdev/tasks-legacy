---
title: "createTable callback is typed as the abstract TableDefinition, not the adapter's"
status: in-progress
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7024
claim: "2026-08-25T01:44:12Z"
assignee: "retire-attribute-set-narrow-to"
blocked-by: null
closed-reason: null
---

## Context

Surfaced by review of PR #7018
(`abstract-schema-creation-declares-mysql-only-visitors` /
`table-definition-enum-bypasses-enum-type-validation`, RFC 0051).

`SchemaStatements#createTable` types its block parameter as the **abstract**
`TableDefinition`:

```ts
// packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:363-364
| ((t: TableDefinition) => void),
fn?: (t: TableDefinition) => void,
```

but the object actually yielded is whatever `createTableDefinition` returns, which
every adapter overrides with its own subclass:

- `postgresql-adapter.ts:3723` → `PgTableDefinition`
- `abstract-mysql-adapter.ts:213` → `MysqlTableDefinition`
- `sqlite3-adapter.ts:208`

This mirrors Rails, where `create_table` yields `create_table_definition(...)`
and `PostgreSQL::TableDefinition` picks up `include PostgreSQL::ColumnMethods`
(`postgresql/schema_definitions.rb:245`, `:181`). Ruby resolves the method on the
yielded object, so `t.citext`/`t.ltree`/`t.enum` just work inside a PG
`create_table` block.

TypeScript resolves against the **declared** parameter type instead, so every
adapter-specific column method is a type error inside a `createTable` callback
even though it runs correctly. That is ~25 names today: the schema dumper emits
`t.citext`, `t.ltree`, `t.timestamptz`, `t.hstore`, `t.inet`, `t.cidr`,
`t.macaddr`, `t.xml`, `t.bitVarying`, `t.money`, `t.int4range`, `t.int8range`,
`t.numrange`, `t.daterange`, `t.tsrange`, `t.tstzrange`, `t.tsvector`,
`t.interval`, `t.oid`, `t.point`, `t.line`, `t.lseg`, `t.box`, `t.enum`, …
(`schema-dumper.ts:205-260`, emitted at `abstract/schema-dumper.ts:320-323`),
and none of those is declared on the abstract `TableDefinition` — `jsonb` is the
only PG name that is, and it is arguably itself misplaced there.

Consequence: a dumped `db/schema.ts` for a PG database does not typecheck, and
test call sites work around it with the established
`(t as PgTableDefinition)` cast (`adapters/postgresql/ltree.test.ts:36`,
`collation.test.ts`, `persistence.test.ts:1884`,
`adapters/postgresql/enum.test.ts`).

PR #7018 removed abstract `TableDefinition#enum` (Rails has no such method — it
is a `PostgreSQL::ColumnMethods` name), which makes `enum` join that set rather
than being the one PG name papered over on the abstract class. It did not widen
the set: the other ~25 were already in it on `main`.

## Converged shape

`createTable` (and `changeTable` / `createJoinTable`, which have the same
`(t: TableDefinition) => void` shape) yield the adapter's own table-definition
type, so a PG `createTable` block type-checks `t.enum` / `t.citext` / `t.ltree`
without a cast, and a dumped `db/schema.ts` typechecks as emitted.

Candidate approach: type the callback against the receiver's own
`createTableDefinition` return — e.g. `fn?: (t: ReturnType<this["createTableDefinition"]>) => void`
— or declare `createTable` on each adapter with its narrowed callback type.
Watch the variance: `fn` is an arrow-typed property, so under
`strictFunctionTypes` a narrower parameter is contravariantly rejected at the
call site; the fix has to live on the declaration, not the caller.

## Acceptance criteria

- A PG `createTable` callback resolves `t.enum(...)` and the other
  `PostgreSQL::ColumnMethods` names with no cast.
- The `(t as PgTableDefinition)` casts in `adapters/postgresql/ltree.test.ts`,
  `collation.test.ts`, `persistence.test.ts` and `adapters/postgresql/enum.test.ts`
  are deleted.
- MySQL and SQLite `createTable` callbacks keep resolving their own adapter's
  names; no adapter-specific name is added to abstract `TableDefinition`
  (that is the deviation RFC 0051 is removing).
- `pnpm typecheck` clean; `parity:api` / `parity:test` deltas non-negative.
