---
title: "converge-postgresql-database-statements-call-set-rows"
status: in-progress
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: 7009
claim: "2026-08-24T22:30:08Z"
assignee: "converge-postgresql-database-statements-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

Two residual `kind: "set"` rows on
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/postgresql/database-statements.json`
with no owning RFC named at all:

| ruby method               | missing call          | Rails                               |
| ------------------------- | --------------------- | ----------------------------------- |
| `last_insert_id_result`   | `internal_exec_query` | `postgresql/database_statements.rb` |
| `returning_column_values` | `first`               | `postgresql/database_statements.rb` |

`last_insert_id_result` is `internal_exec_query("SELECT currval(...)")`; trails
reads the sequence value through `queryValue`, which is `internalExecQuery` plus
`.rows.first.first` — the row extraction is folded in, so the published call-set
never records the delegation. Converging means calling `internalExecQuery`
directly and doing the extraction at Rails' site.

`returning_column_values -> first` is `result.rows.first` — `Array#first` on a JS
array is `[0]`, and `first` is not a ported method name. That is a strong
PERMANENT `@missingRailsCall` candidate; it is called out here rather than left
unowned so the receipt gets written once, deliberately.

## Acceptance criteria

- [ ] `lastInsertIdResult` calls `internalExecQuery` at Rails' call site.
- [ ] `returningColumnValues` carries a `@missingRailsCall first — PERMANENT: …`
      receipt, or converges if a ported `first` receiver spelling exists by then.
- [ ] Both rows deleted by hand; `pnpm parity:api:calls` / `:args` green; no
      `--write`, no reseed.
- [ ] PostgreSQL lane green (plus SQLite and MySQL/MariaDB).
