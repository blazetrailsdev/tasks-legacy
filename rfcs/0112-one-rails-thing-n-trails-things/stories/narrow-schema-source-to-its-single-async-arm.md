---
title: "SchemaSource advertises a sync arm Rails' @connection does not have"
status: in-progress
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7060
claim: "2026-08-25T18:47:56Z"
assignee: "converge-association-check-klass-onto-reflection-check-validity"
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing #7051 (`converge-schema-dumper-tables-single-body`).
That PR collapsed `SchemaDumper#tables` and `#dump` onto Rails' single bodies
(`schema_dumper.rb:60-68`, `:134-155`), so **every** dumper entry point is now
`await`-driven. The `SchemaSource` interface it reads through still advertises a
synchronous arm on each member that nothing takes any more:

`packages/activerecord/src/schema-dumper.ts` (interface `SchemaSource`):

```ts
tables(): string[] | Promise<string[]>;
columns(tableName: string, pkNames?: readonly string[]): ColumnInfo[] | Promise<ColumnInfo[]>;
indexes(tableName: string): IndexInfo[] | Promise<IndexInfo[]>;
```

and `fetchTableOptions` likewise returns
`Record<string, unknown> | Promise<Record<string, unknown>>`.

Rails has no such union, and no `SchemaSource` at all: `SchemaDumper` holds
`@connection` (`schema_dumper.rb:113` `@connection = connection`) and calls
`@connection.tables` / `.columns(table)` / `.indexes(table)` directly
(`:135`, `:158`, `schema_dumper.rb` `indexes_in_create`). The connection is
always the real adapter, so there is exactly one arm.

The union's only remaining consumer was the synchronous fast path in `tables()`
that #7051 deleted. What is left is a type-level invitation to reintroduce a
sync branch — the same "one Rails thing, N trails things" shape RFC 0112 exists
to burn down — plus the `void | Promise<void>` returns on the `schemas` /
`extensions` / `types` / `virtualTables` hooks, which every subclass already
implements as `async`.

Related, already-filed, and deliberately NOT duplicated here:
`adapter-schema-source-column-flag-duck-typing` (0023) covers
`AdapterSchemaSource` hand-projecting column flags and the `_fkHookHost` /
`_hookHost` duck-types standing in for Rails' `supports_foreign_keys?`
(`schema_dumper.rb:145`). This story is only about the sync/async union.

## Converged shape

Narrow every `SchemaSource` member and every dumper hook to its async arm —
`Promise<string[]>`, `Promise<ColumnInfo[]>`, `Promise<IndexInfo[]>`,
`Promise<Record<string, unknown>>`, `Promise<void>` — matching the single arm
Rails' `@connection` presents. The remaining plain-object sources in
`schema-dumper.trails.test.ts` that still declare sync members are widened to
`async` at the same time (they already flow through the async path; only their
declarations lag).

Rails cite: `schema_dumper.rb:113` (`@connection`), `:135`, `:145`, `:158`.

## Acceptance criteria

- [ ] `SchemaSource`'s `tables` / `columns` / `indexes` / `fetchTableOptions`
      declare a single `Promise<...>` arm; no `T | Promise<T>` union remains on
      the interface.
- [ ] The `schemas` / `extensions` / `types` / `virtualTables` hooks declare
      `Promise<void>`, not `void | Promise<void>`.
- [ ] No sync branch or `instanceof Promise` fork is reintroduced anywhere in
      `schema-dumper.ts`.
- [ ] `schema-dumper.test.ts`, `schema-dumper.trails.test.ts`,
      `connection-adapters/abstract/schema-dumper.trails.test.ts` and
      `adapters/sqlite3/virtual-table.test.ts` stay green.
