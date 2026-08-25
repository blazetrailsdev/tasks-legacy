---
title: "Migration#createTable callback is typed as the abstract TableDefinition, not the connection's"
status: draft
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
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

## Context

Surfaced while converging `create-table-callback-yields-abstract-table-definition`
(PR #7024, RFC 0051), which made the ADAPTER-level `createTable` yield the
receiver's own table-definition type via `TableDefinitionOf<this>`
(`connection-adapters/abstract/schema-definitions.ts`). That removed every
`(t as PgTableDefinition)` cast in the adapter test files.

One cast survived, and it is the Migration-level surface rather than the
adapter one:

```ts
// packages/activerecord/src/adapters/postgresql/enum.test.ts:339
await Schema.define(async (schema) => {
  await schema.createTable("postgresql_enums_in_test_schema", { force: "cascade" }, (t) => {
    (t as PgTableDefinition).enum("current_mood", { enum_type: "mood_in_test_schema" });
  });
});
```

`Migration#createTable` (`packages/activerecord/src/migration.ts:517`) types its
block parameter as the abstract `TableDefinition`, and `Migration` has no
`createTableDefinition` of its own for `TableDefinitionOf<this>` to resolve
against — it forwards to the connection. So the conditional type falls through
to its `TableDefinition` default and every adapter-specific column method is a
type error inside a `Schema.define` / migration `create_table` block.

Rails has no such problem for the same reason it had none at the adapter level:
`ActiveRecord::Migration#method_missing` forwards to the connection
(`activerecord/lib/active_record/migration.rb:1024-1036`), and Ruby resolves
the yielded object's methods at call time, so `t.enum` / `t.citext` / `t.ltree`
just work inside a migration's `create_table` block.

## Converged shape

`Migration#createTable` (and `changeTable` / `createJoinTable`, which share the
`(t: TableDefinition) => void` shape) resolve their callback parameter against
the connection they forward to, so a PG migration block type-checks
`t.enum(...)` with no cast — the same fidelity fix PR #7024 made one layer down.

The mechanism already exists: `TableDefinitionOf<A>` in
`connection-adapters/abstract/schema-definitions.ts`. What is missing is a way
for `Migration` to name the adapter type it forwards to. Watch the variance
note from #7024: `fn` is an arrow-typed property, so under
`strictFunctionTypes` a narrower parameter is contravariantly rejected at the
call site — the fix has to live on the declaration, not the caller.

## Acceptance criteria

- A PG `Schema.define` / migration `createTable` callback resolves `t.enum(...)`
  and the other `PostgreSQL::ColumnMethods` names with no cast.
- The surviving `(t as PgTableDefinition)` cast at
  `packages/activerecord/src/adapters/postgresql/enum.test.ts:339` is deleted,
  along with its now-unused `PgTableDefinition` import if nothing else uses it.
- MySQL and SQLite migration callbacks keep resolving their own adapter's names;
  no adapter-specific name is added to abstract `TableDefinition` (that is the
  deviation RFC 0051 removed in #7024 and #7018).
- `pnpm typecheck` clean; `parity:api` / `parity:test` deltas non-negative.
