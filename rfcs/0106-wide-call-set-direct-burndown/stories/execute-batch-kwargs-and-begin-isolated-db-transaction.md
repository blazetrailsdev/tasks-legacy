---
title: "executeBatch kwargs travel; beginIsolatedDbTransaction ports its execute_batch call"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: "2026-08-23T13:12:30Z"
assignee: "converge-enable-query-cache-onto-the-block-value-return"
blocked-by: null
closed-reason: null
---

## Context

Rails' `execute_batch` takes `**kwargs` at every level
(`abstract/database_statements.rb:594`, `postgresql/:195`, `sqlite3/:126`,
`mysql2/:17`) and `AbstractMysqlAdapter#begin_isolated_db_transaction`
(`abstract_mysql_adapter.rb:230-239`) is the caller that depends on it:

    execute_batch(
      ["SET TRANSACTION ISOLATION LEVEL #{...}", "BEGIN"],
      "TRANSACTION",
      allow_retry: true,
      materialize_transactions: false,
    )

In trails the kwargs object exists only on the PostgreSQL override (added by
PR #6910). The declared signatures —
`connection-adapters/abstract-adapter.ts:703` and
`connection-adapters/abstract/database-statements.ts:154` — are
`executeBatch(statements, name?)`, so no caller can pass the kwargs, and
`AbstractMysqlAdapter#beginIsolatedDbTransaction`
(`abstract-mysql-adapter.ts:554-556`) is a `void isolation` no-op rather than
the `execute_batch` call above.

## Acceptance criteria

- [ ] `executeBatch`'s declared signature carries the kwargs
      (`allowRetry`, `materializeTransactions`) on the abstract-adapter
      interface, the DatabaseStatements interface, and every adapter override.
- [ ] `beginIsolatedDbTransaction` ports `abstract_mysql_adapter.rb:230-239`
      verbatim, passing `allowRetry: true, materializeTransactions: false`.
- [ ] `pnpm parity:api:calls` / `:args` green; MySQL lane green.
