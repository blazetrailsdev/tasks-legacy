---
title: "Converge the residual adapter call-argument locals (PG default/rename/index, SQLite3 table_info and dflt_value)"
status: done
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 7024
claim: "2026-08-25T01:44:12Z"
assignee: "retire-attribute-set-narrow-to"
blocked-by: null
closed-reason: null
---

## Context

Residual `naming` rows read off `pnpm parity:api:calls:args:report` while
working #7014. Each is a local or parameter that dropped the Rails identifier,
which CLAUDE.md's "Locals and parameters" rule says must be kept:

- `connection-adapters/postgresql-adapter.ts` — `extract_default_function` calls
  `has_default_function?(default_value, default)`
  (`activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb`),
  trails passes `defaultExpr`; `rename_table` calls
  `pk_and_sequence_for(new_name)`, trails passes `renamedName`; `remove_index`
  calls `quote_table_name(index_to_remove)`, trails passes `toString`.
- `connection-adapters/sqlite3-adapter.ts` — `table_info` calls
  `quote_table_name(table_name)`
  (`.../sqlite3_adapter.rb`), trails passes `schema` / `bare`.
- `connection-adapters/sqlite3/schema-statements.ts` — `new_column_from_field`
  calls `extract_value_from_default(default)` and
  `extract_default_function(default_value, default)`
  (`.../sqlite3/schema_statements.rb`), trails passes `dfltValue`.

These are report-only rows (`class: "naming"`), so no gate is red — they are
straight identifier convergence.

## Acceptance criteria

1. Each local/parameter above carries the Rails identifier, camelCased.
2. The rows are gone from `pnpm parity:api:calls:args:report`; no new
   `call-mismatches-exclude/` row.
3. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green; SQLite
   and PostgreSQL lanes green.
