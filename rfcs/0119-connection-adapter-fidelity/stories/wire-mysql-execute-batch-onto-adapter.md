---
title: "Wire mysql2 executeBatch onto the MySQL adapter class"
status: draft
updated: 2026-08-03
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Mysql2::DatabaseStatements#execute_batch`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql2/database_statements.rb:17-21`)
is ported at
`packages/activerecord/src/connection-adapters/mysql2/database-statements.ts:160`
but the function is **never assigned to any adapter class** —
`grep executeBatch packages/activerecord/src/connection-adapters/mysql2-adapter.ts
packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts` returns
nothing. Its only consumers are its own module and the abstract-adapter interface
declaration (`abstract-adapter.ts:612`). PostgreSQL wires its copy
(`postgresql-adapter.ts:2310 executeBatch = pgExecuteBatch`); MySQL does not.

Discovered while shipping #5957, which made `combineMultiStatements` async so
`executeBatch` resolves the real server `max_allowed_packet`. That fix is
correct but currently unreachable on a live MySQL adapter, because
`executeBatch` itself is not on the class.

## Acceptance criteria

- `Mysql2Adapter` (or `AbstractMysqlAdapter`, matching where Rails' module is
  included) assigns `executeBatch` from `mysql2/database-statements.ts`.
- A test exercises the batch path on the MySQL adapter and asserts the
  statements are split against the server-reported `max_allowed_packet`.
- Verify the mysql2 adapter's multi-statement flag handling
  (`multiStatementsEnabled`) is consistent with the wired path.
