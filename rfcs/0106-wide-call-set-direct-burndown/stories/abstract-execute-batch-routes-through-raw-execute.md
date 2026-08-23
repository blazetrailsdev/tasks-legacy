---
title: "abstract executeBatch calls rawExecute per statement and forwards name"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: "2026-08-23T13:12:30Z"
assignee: "converge-enable-query-cache-onto-the-block-value-return"
blocked-by: null
closed-reason: null
---

## Context

`abstract/database_statements.rb:594-598`:

    def execute_batch(statements, name = nil, **kwargs)
      statements.each do |statement|
        raw_execute(statement, name, **kwargs)
      end
    end

trails' `executeBatch`
(`packages/activerecord/src/connection-adapters/abstract/database-statements.ts:1772-1791`)
loops `executeMutation(statement)` — dropping both the `name` and the kwargs —
and sets `_inQueryTransformers` around each statement so `preprocessQuery`
skips the transformer pass. Rails needs no such flag because `raw_execute` is
below `preprocess_query` (`internal_execute`'s step, `:589-591`).

PR #6910 converged the PostgreSQL override
(`postgresql/database_statements.rb:195-197`) to a single `rawExecute`;
sqlite3 was already converged. The abstract base is the remaining
`_inQueryTransformers` loop besides mysql2.

## Acceptance criteria

- [ ] Abstract `executeBatch` calls `rawExecute(statement, name, [], false,
false, allowRetry, materializeTransactions)` per statement, matching
      `abstract/database_statements.rb:596`.
- [ ] `name` is forwarded (today it is dropped).
- [ ] The `_inQueryTransformers` flag no longer wraps the batch path; if no
      caller remains, retire the flag and its `preprocessQuery` arm.
- [ ] Baseline rows for `execute_batch` in the abstract shard deleted, not
      reworded; `pnpm parity:api:calls` / `:args` green on all three lanes.
