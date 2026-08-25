---
title: "create_table_definition is ported twice for MySQL"
status: done
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 80
priority: 34
pr: 7019
claim: "2026-08-25T00:18:05Z"
assignee: "move-ts-only-extras-out-of-mirrored-activemodel-translation-test-file"
blocked-by: null
closed-reason: null
---

## Context

Rails defines `create_table_definition(name, **options)` once, on the
`MySQL::SchemaStatements` module
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/schema_statements.rb:172`),
as `MySQL::TableDefinition.new(self, name, **options)`.

trails has it twice after PR #5944:

- `packages/activerecord/src/connection-adapters/mysql/schema-statements.ts`
  — a `this: VisitorHostAdapter`-typed function in the file that maps onto
  `mysql/schema_statements.rb`, exercised only by its own unit test.
- `packages/activerecord/src/connection-adapters/abstract-mysql-adapter.ts:223`
  — the live one, an `AbstractMysqlAdapter` method that every
  `SchemaStatements#createTable` / `changeColumn` path actually calls; it also
  strips a caller-supplied `adapter` / `adapterName` before substituting `this`.

Two ports of one Rails method, only one of them reachable. The related
`converge-schema-statements-companion-onto-mixin` story covers the broader
companion-class deviation; this one is the narrow MySQL duplicate.

## Acceptance criteria

- `create_table_definition` exists once, in the file that maps onto
  `mysql/schema_statements.rb`, wired into `AbstractMysqlAdapter` by the
  CLAUDE.md module-mixin convention.
- The `adapter` / `adapterName` strip either lives with it or is shown to be
  unnecessary once the abstract caller threads the adapter itself.
- parity:api activerecord matched-method count does not decrease; novel count
  does not increase.
