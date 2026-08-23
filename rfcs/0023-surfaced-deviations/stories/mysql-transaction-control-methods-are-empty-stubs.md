---
title: "AbstractMysqlAdapter's five transaction-control methods are empty stubs; exec_restart_db_transaction is an unimplemented no-op behind a true supports? flag"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced reading `abstract-mysql-adapter.ts` end-to-end for RFC 0106 wave 3b
(PR #6577).

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:227-252`
gives every transaction-control method a real body:

    def begin_db_transaction
      internal_execute("BEGIN", "TRANSACTION", allow_retry: true, materialize_transactions: false)
    end

    def commit_db_transaction
      internal_execute("COMMIT", "TRANSACTION", allow_retry: false, materialize_transactions: true)
    end

    def exec_rollback_db_transaction
      internal_execute("ROLLBACK", "TRANSACTION", allow_retry: false, materialize_transactions: true)
    end

    def exec_restart_db_transaction
      internal_execute("ROLLBACK AND CHAIN", "TRANSACTION", allow_retry: false, materialize_transactions: true)
    end

trails (`connection-adapters/abstract-mysql-adapter.ts:543-553`) has five EMPTY
bodies:

    async beginDbTransaction(): Promise<void> {}
    async beginIsolatedDbTransaction(isolation: string): Promise<void> { void isolation; }
    async commitDbTransaction(): Promise<void> {}
    async execRollbackDbTransaction(): Promise<void> {}
    async execRestartDbTransaction(): Promise<void> {}

Three of the five are shadowed by real overrides on `Mysql2Adapter`
(`beginDbTransaction`:1045, `beginIsolatedDbTransaction`:1079,
`commitDbTransaction`:1110), so the empty bodies are dead weight there — but
they are still stubs Rails does not have, sitting on the class Rails puts the
real bodies on.

The other two are NOT shadowed:

- **`execRestartDbTransaction` has no implementation anywhere.** `grep -rn
"ROLLBACK AND CHAIN" packages/` matches only `postgresql-adapter.ts:1972`.
  Meanwhile `supportsRestartDbTransaction()` returns `true`
  (abstract-mysql-adapter.ts:376, mirroring rb:112-114), so a caller that asks
  MySQL to restart a transaction is told yes and then gets a silent no-op.
- **`execRollbackDbTransaction`** is empty, and the real MySQL rollback body
  lives at `mysql2-adapter.ts:1124` under the name `rollbackDbTransaction`.
  Rails' `rollback_db_transaction` is the PUBLIC wrapper in
  `DatabaseStatements` (`abstract/database_statements.rb`); the adapter hook
  Rails asks adapters to define is `exec_rollback_db_transaction`. So the body
  is on the wrong name.

## Converged shape

`beginDbTransaction`, `beginIsolatedDbTransaction`, `commitDbTransaction`,
`execRollbackDbTransaction` and `execRestartDbTransaction` each carry Rails'
`internalExecute(...)` body ON `AbstractMysqlAdapter`, with Rails' `allowRetry`
/ `materializeTransactions` kwargs, and the `Mysql2Adapter` overrides are
retired unless they carry driver-specific state the base cannot reach. The
`rollbackDbTransaction` body moves to `execRollbackDbTransaction`.

Coordinate with RFC 0076 (execute-primitive-convergence), which owns
`internal_execute` on this file.

## Acceptance criteria

- [ ] No empty transaction-control body remains on `AbstractMysqlAdapter`.
- [ ] `execRestartDbTransaction` emits `ROLLBACK AND CHAIN`, or
      `supportsRestartDbTransaction()` stops claiming support — the two must agree.
- [ ] `execRollbackDbTransaction` holds the rollback body under the Rails name.
- [ ] A regression test that FAILS on baseline covers the restart path on MySQL/MariaDB.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.

## Update after PR #6913 (2026-08-23)

One of the five is no longer a stub. `beginIsolatedDbTransaction` now ports
`abstract_mysql_adapter.rb:230-239` in place of its `void isolation` no-op,
including the Rails source comment and the `transaction_isolation_levels.fetch`
KeyError arms:

    await this.executeBatch([`SET TRANSACTION ISOLATION LEVEL ${level}`, "BEGIN"], "TRANSACTION", {
      allowRetry: true,
      materializeTransactions: false,
    });

That was possible because #6913 put the `allow_retry` /
`materialize_transactions` kwargs on `executeBatch`'s declared signature at
every level (`abstract_mysql_adapter.rb:230-239` is the caller Rails' kwargs
exist for).

So the remaining scope here is **four** empty bodies, not five, plus a new
sub-item this surfaced:

- **`Mysql2Adapter`'s `beginIsolatedDbTransaction` override
  (`mysql2-adapter.ts`) is now redundant divergence and should be retired.**
  Rails defines `begin_isolated_db_transaction` ONLY on `AbstractMysqlAdapter`
  — `mysql2_adapter.rb` and `mysql2/database_statements.rb` have no such
  method — so the override is invented surface. It hand-rolls
  `withRawConnection({ allowRetry: true, materializeTransactions: false })` +
  two `internalExecute` calls + a `this._inTransaction = true` assignment,
  which is what the now-ported abstract body expresses through `executeBatch`
  → `rawExecute` → `withRawConnection`. Retiring it needs the `_inTransaction`
  bookkeeping accounted for (that flag is trails-only) and the MySQL lane
  green; it shadows the abstract method today, so the abstract port is
  currently unexercised on mysql2.
