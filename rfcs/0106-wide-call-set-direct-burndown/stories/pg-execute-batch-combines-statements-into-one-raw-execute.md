---
title: "pg-execute-batch-combines-statements-into-one-raw-execute"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6910
claim: "2026-08-23T12:27:28Z"
assignee: "pg-execute-batch-combines-statements-into-one-raw-execute"
blocked-by: null
closed-reason: null
---

## Context

`postgresql/database_statements.rb:195-197`:

    def execute_batch(statements, name = nil, **kwargs)
      raw_execute(combine_multi_statements(statements), name, batch: true, **kwargs)
    end

trails' `executeBatch`
(`packages/activerecord/src/connection-adapters/postgresql/database-statements.ts:273-292`)
loops the statements and calls `this.execute(statement, [], name)` per statement,
setting `_inQueryTransformers` around each so `preprocessQuery` skips the
transformer pass. Two rows stay in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/postgresql/database-statements.json`
(`execute_batch | combine_multi_statements`, `execute_batch | raw_execute`).

RFC 0106 wave 5g left both baselined rather than mint receipts: routing through
`execute` instead of `rawExecute` is a real divergence — PG accepts multiple
statements in one simple-query message, which is exactly what
`combine_multi_statements` exists for, so the loop is not a language shortcoming.

## Acceptance criteria

- [ ] `executeBatch` joins the statements through `combineMultiStatements` and
      issues ONE `rawExecute(..., { batch: true, ...kwargs })`, matching
      `database_statements.rb:196`, with the `materializeTransactions` /
      `allowRetry` kwargs travelling on that call.
- [ ] The `_inQueryTransformers` flag no longer spans a loop (see
      project_inquerytransformers_flag_leaks_when_rerouted_off_preprocessquery).
- [ ] Both baseline rows deleted, not reworded.
- [ ] `pnpm parity:api:calls` / `:args` green; PostgreSQL lane green.
