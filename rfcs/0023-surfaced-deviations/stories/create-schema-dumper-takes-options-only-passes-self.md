---
title: "create_schema_dumper takes (source, options) where Rails takes (options) and passes self"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in RFC 0096 wave 5 (PR #6917) as three `module-mixin-receiver` rows
that are not receiver rows at all — they are an arity deviation, so the mixin
rewiring the RFC prescribes cannot close them.

Rails takes ONE argument and passes `self`:

- `activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:1541-1543`
  — `def create_schema_dumper(options); SchemaDumper.create(self, options); end`
- `activerecord/lib/active_record/connection_adapters/postgresql/schema_statements.rb:884-886`
  — `PostgreSQL::SchemaDumper.create(self, options)`
- `activerecord/lib/active_record/connection_adapters/sqlite3/schema_statements.rb:122-124`
  — `SQLite3::SchemaDumper.create(self, options)`
- `activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb:107-109`
  — `MySQL::SchemaDumper.create(self, options)`

trails takes TWO, `(source, options)`:

- `packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:1934`
- `packages/activerecord/src/connection-adapters/postgresql-adapter.ts:3920`
- `packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:1699`
- `packages/activerecord/src/connection-adapters/sqlite3/schema-statements.ts:202`
- `packages/activerecord/src/connection-adapters/mysql2-adapter.ts:1279`

The blocker is the single call site,
`packages/activerecord/src/schema-dumper.ts:572-573`:

```ts
if (isDatabaseAdapter(source) && typeof (source as any).createSchemaDumper === "function") {
  dumper = (source as any).createSchemaDumper(wrappedSource, {}) as SchemaDumper;
```

It deliberately passes a **wrapped** source, not the adapter, so simply dropping
the parameter and using `this` would change what the dumper reads. Converging
therefore means understanding why the wrapper exists and either folding it into
the adapter or moving it to where Rails puts it — not a mechanical parameter
drop. `schema-dumper.ts:543` reads the same method off the pool for the same
reason.

## Converged shape

`createSchemaDumper(options)` on every adapter, passing `this` to
`SchemaDumper.create`, with `schema-dumper.ts` obtaining its wrapped source some
other way (or not needing one).

## Acceptance criteria

1. All five `createSchemaDumper` definitions take `(options)` alone and pass
   `this`, matching the four Ruby definitions cited above.
2. `schema-dumper.ts:543` and `:572-573` are converged with them; whatever
   `wrappedSource` provides is either unnecessary or supplied without an extra
   parameter, with the reason recorded at the call site.
3. The three `create_schema_dumper` rows leave the convergeable `naming`
   population; no `call-mismatches-exclude/` row is added.
4. `pnpm build && pnpm test` green — schema-dump output byte-identical on
   SQLite, PostgreSQL and MySQL; `pnpm parity:api` arity delta non-negative.
