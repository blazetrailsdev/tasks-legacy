---
title: "mysql2 executeBatch calls rawExecute per combined block, not execute"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 6913
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

## Status after PR #6913 — body converged, install blocked on the driver

PR #6913 converged the ported body: `executeBatch`
(`packages/activerecord/src/connection-adapters/mysql2/database-statements.ts`)
now calls `rawExecute(statement, name, [], false, false, allowRetry,
materializeTransactions, true)` per combined block, matching
`mysql2/database_statements.rb:17-21`, and its `@missingRailsCall raw_execute`
tag is deleted. The `_inQueryTransformers` batch-suppression arm is retired
repo-wide.

What remains is the **install**: `Mysql2Adapter` does not assign
`executeBatch` beside `performQuery`, so a real mysql2 connection still
inherits `AbstractAdapter`'s per-statement loop. Rails installs it via
`include Mysql2::DatabaseStatements` (`mysql2_adapter.rb:21`).

Installing it today breaks every batch on the MySQL lane. Measured against
`mariadb:11` with the override wired:

    ER_PARSE_ERROR (1064): You have an error in your SQL syntax ... near
    'CREATE TABLE batch_probe (id int)' at line 2
    sql: 'DROP TABLE IF EXISTS batch_probe;\nCREATE TABLE batch_probe (id int)'

`combine_multi_statements` (`mysql/database_statements.rb:66-76`) joins with
`";\n"` unconditionally and carries no multi-statement guard. In Rails the
guard is in `perform_query`, which turns the option on for the batch query
itself and resets it after:

    reset_multi_statement = if batch && !multi_statements_enabled?
      raw_connection.set_server_option(::Mysql2::Client::OPTION_MULTI_STATEMENTS_ON)
      true
    end

(`mysql2/database_statements.rb:41-45`). node-mysql2 exposes multi-statements
only as the `multipleStatements` connection-creation option and ships **no
command class for `COM_SET_OPTION`** — `lib/constants/commands.js:31` defines
`SET_OPTION: 0x1b` but `lib/commands/` has no `set_option.js`, and the option
is protocol-level with no SQL equivalent. trails already ports the predicate
(`isMultiStatementsEnabled`, mirroring `mysql2/database_statements.rb:31-39`);
it is unused because nothing can act on a false answer.

### Remaining acceptance criteria

- [ ] `Mysql2Adapter` installs `executeBatch as mysql2ExecuteBatch` beside
      `performQuery`, and the install-site note added by #6913 is deleted.
- [ ] `performQuery`'s `batch` arm ports `mysql2/database_statements.rb:41-45`
      and its reset, or documents at the call site why the chosen mechanism is
      the only one node-mysql2 allows. Enabling `multipleStatements` globally on
      the pool config is NOT acceptable without its own decision: Rails scopes
      the option to the batch query precisely to limit the injection surface,
      and a global enable changes behavior for every existing mysql2 user.
- [ ] The `ER_PARSE_ERROR` repro above passes: a two-statement `executeBatch`
      succeeds on the MariaDB lane.
- [ ] MariaDB and MySQL lanes green.
