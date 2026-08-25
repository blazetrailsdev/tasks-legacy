---
title: "PostgreSQL's dropTable override has no Rails counterpart"
status: draft
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# PostgreSQL's `dropTable` override has no Rails counterpart

## Context

Surfaced in PR #7038 (RFC 0051, `migration-recording-flag-should-be-the-connection`).

Rails defines `drop_table` twice: the base
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_statements.rb:540`)
and one MySQL override
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb`,
for `TEMPORARY`). PostgreSQL has NO `drop_table` — `grep -n "def drop_table"
vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql*`
returns nothing; PG rides the base, which already emits
`DROP TABLE#{' IF EXISTS'} ... #{' CASCADE'}`.

trails has a third body,
`packages/activerecord/src/connection-adapters/postgresql/schema-statements-class.ts`
(`override async dropTable`), which re-implements the base line for line: the
same trailing-options parse, the same `tableNames.length === 0` ArgumentError,
the same `clearDataSourceCacheBang` loop, the same SQL. The only difference is
that it quotes all names into one statement.

Because the parse is copied rather than inherited, a change to the base has to
be made three times. PR #7038 hit exactly that: teaching `dropTable` to ignore
Ruby's `&block` (passed as a trailing function once `Migration` routes schema
statements through the connection) had to be repeated in all three bodies, and
the PG one was missed first — reddening
`invertible-migration.test.ts` > `migrate down with table name prefix` on the
PostgreSQL lanes only, with `table "undefined" does not exist`.

## Converged shape

Delete `PostgreSQL::SchemaStatements#dropTable` and let PG inherit the base, as
Rails does. If the single-statement multi-table `DROP TABLE a, b` form is worth
keeping, it belongs in the base (Rails' base loops one statement per table), not
in a PG-only copy.

## Acceptance criteria

- [ ] No `dropTable` in `postgresql/schema-statements-class.ts`; PG resolves the
      inherited one.
- [ ] The `dropTable` arg parse exists in the base and the MySQL override only,
      matching Rails' two definitions.
- [ ] `invertible-migration.test.ts`, `schema-statements-class.trails.test.ts`
      and `schema-statements.trails.test.ts` stay green on SQLite, PostgreSQL
      and MySQL/MariaDB.
