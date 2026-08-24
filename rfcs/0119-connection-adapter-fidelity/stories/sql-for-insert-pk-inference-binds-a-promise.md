---
title: "sqlForInsert's pk-inference branch binds the async primaryKey Promise into RETURNING"
status: draft
updated: 2026-07-29
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

`sqlForInsert`
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:2176-2201`)
ports Rails' pk-inference branch
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/database_statements.rb:702-713`):
when `pk` is nil it extracts the table from the INSERT SQL and asks the adapter
for the primary key.

In Rails `primary_key(table_ref)` is synchronous. In trails the adapter's
`primaryKey(tableName)` returns `Promise<string | string[] | null>`
(`abstract-adapter.ts:361`), so `resolvedPk = this.primaryKey?.(tableRef) ?? null`
binds a **Promise**. It is non-null, so `returningColumns` becomes `[Promise]` and
the emitted SQL is `RETURNING "[object Promise]"`.

The branch is only reachable when the caller passes no `pk`, and the live callers
all pass one, which is why it has never fired. PR 5585 made the MySQL family take
this code path for the first time (MariaDB >= 10.5 `supports_insert_returning?`),
so the latent bug is now one nil `pk` away on three adapters instead of two.

Note `DatabaseStatementsHost.primaryKey?(table)` is typed `string | null`
(database-statements.ts:174) while the real adapter method is async — the host
interface hides the mismatch from the compiler.

## Acceptance criteria

- The pk-inference path resolves the primary key before building the RETURNING
  list, or the host interface is corrected so the async return is a type error
  rather than silently stringified.
- A regression test drives `execInsert`/`sqlForInsert` with `pk` omitted on an
  adapter whose `supports_insert_returning?` is true and asserts the emitted
  RETURNING names the table's real primary key. It must fail on baseline.
- Composite primary keys (`primaryKey` may return `string[]`) either work or
  raise, not stringify.
