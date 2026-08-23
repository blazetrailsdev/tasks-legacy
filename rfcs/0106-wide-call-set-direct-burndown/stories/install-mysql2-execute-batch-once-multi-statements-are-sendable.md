---
title: "install-mysql2-execute-batch-once-multi-statements-are-sendable"
status: closed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Superseded: folded back into mysql2-execute-batch-routes-through-raw-execute, which stays open (blocked) rather than being closed by PR #6913. Two stories for one gap is duplicate debt; the driver evidence now lives on the original."
---

## Context

`Mysql2::DatabaseStatements#execute_batch` is ported
(`packages/activerecord/src/connection-adapters/mysql2/database-statements.ts`,
converged in #6913 to `rawExecute(statement, name, [], false, false, allowRetry,
materializeTransactions, true)` per combined block, matching
`mysql2/database_statements.rb:17-21`) but it is **not installed onto
`Mysql2Adapter`**, so a real mysql2 connection still inherits
`AbstractAdapter`'s per-statement loop and never passes `batch: true`.

Wiring it as-is is provably broken. Measured against `mariadb:11`:

    ER_PARSE_ERROR (1064): You have an error in your SQL syntax ... near
    'CREATE TABLE batch_probe (id int)' at line 2
    sql: 'DROP TABLE IF EXISTS batch_probe;\nCREATE TABLE batch_probe (id int)'

`combine_multi_statements` (`mysql/database_statements.rb:66-76`) joins with
`";\n"` unconditionally — it carries no multi-statement guard. In Rails the
guard lives in `perform_query`, which turns the option on for the batch query
itself and resets it after:

    reset_multi_statement = if batch && !multi_statements_enabled?
      raw_connection.set_server_option(::Mysql2::Client::OPTION_MULTI_STATEMENTS_ON)
      true
    end

(`mysql2/database_statements.rb:41-45`). node-mysql2 exposes multi-statements
only as the `multipleStatements` connection-creation option and ships **no
command class for `COM_SET_OPTION`** — `lib/constants/commands.js:31` defines
`SET_OPTION: 0x1b` but `lib/commands/` has no `set_option.js` — so the
per-query toggle is unsendable through the driver's public API.

trails already ports the predicate (`isMultiStatementsEnabled`,
mysql2/database-statements.ts, mirroring
`mysql2/database_statements.rb:31-39`); it is unused because nothing can act
on a false answer.

## Acceptance criteria

- [ ] `Mysql2Adapter` installs `executeBatch as mysql2ExecuteBatch` beside
      `performQuery`, and the install-site note added by #6913 explaining why it
      was withheld is deleted.
- [ ] `performQuery`'s `batch` arm ports
      `mysql2/database_statements.rb:41-45` + its reset, or documents at the
      call site why the chosen mechanism is the only one node-mysql2 allows.
      Enabling `multipleStatements` globally on the pool config is NOT
      acceptable without its own decision: Rails scopes the option to the batch
      query, and a global enable widens the injection surface of every query.
- [ ] The `ER_PARSE_ERROR` repro above passes: a two-statement
      `executeBatch` succeeds on the MariaDB lane.
- [ ] MariaDB and MySQL lanes green.
