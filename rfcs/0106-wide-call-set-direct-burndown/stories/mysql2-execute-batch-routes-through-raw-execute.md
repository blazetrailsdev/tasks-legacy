---
title: "mysql2 executeBatch calls rawExecute per combined block, not execute"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-23T13:12:30Z"
assignee: "converge-enable-query-cache-onto-the-block-value-return"
blocked-by: null
closed-reason: null
---

## Context

`mysql2/database_statements.rb:17-21`:

    def execute_batch(statements, name = nil, **kwargs)
      combine_multi_statements(statements).each do |statement|
        raw_execute(statement, name, batch: true, **kwargs)
      end
    end

trails' `executeBatch`
(`packages/activerecord/src/connection-adapters/mysql2/database-statements.ts:184-192`)
loops the combined blocks but calls `this.execute(statement, name)` instead of
`rawExecute(statement, name, ..., batch: true, ...kwargs)`, so the batch
re-enters `preprocessQuery` and needs the `_inQueryTransformers` suppression
flag that Rails does not have. PR #6910 converged the PostgreSQL twin
(`postgresql/database_statements.rb:195-197`) to a single `rawExecute` and
deleted its two baseline rows; sqlite3
(`sqlite3/database-statements.ts:389-402`) was already converged. MySQL is the
last of the three still on the `execute` loop.

## Acceptance criteria

- [ ] mysql2 `executeBatch` calls `rawExecute(statement, name, [], false, false,
allowRetry, materializeTransactions, true)` per combined block, matching
      `mysql2/database_statements.rb:19`.
- [ ] The `_inQueryTransformers` flag no longer wraps the batch path.
- [ ] Any `execute_batch` rows in
      `scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/mysql2/`
      deleted, not reworded; `pnpm parity:api:calls` / `:args` green.
- [ ] MySQL lane green.
