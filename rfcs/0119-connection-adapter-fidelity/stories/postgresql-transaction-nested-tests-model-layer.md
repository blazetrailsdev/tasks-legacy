---
title: "postgresql-transaction-nested-tests-model-layer"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
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
closed-reason: null
---

# Converge adapters/postgresql/transaction-nested tests from adapter-level SQL to the model layer

## Context

Split out of `abstract-mysql-concurrency-tests-model-layer` (RFC 0119), which
converged the two MySQL siblings —
`packages/activerecord/src/adapters/abstract-mysql-adapter/count-deleted-rows-with-lock.test.ts`
and `nested-deadlock.test.ts` — to the model layer. Both now drive
`Sample.transaction(fn, { requiresNew: true })` + `lockBang()` + `update()` over
two real connections, using `withExecutionContext()`
(`packages/activerecord/src/connection-adapters/abstract/connection-pool/execution-context.ts`)
as the `Thread.new` analogue: the pool leases per execution context
(`connection_pool.rb:711` `connection_lease`), so two contexts get two
connections and really deadlock. `IsolatedExecutionState.run` does NOT work —
verified: both sides get the same leased connection and the test hangs.

The PG sibling `packages/activerecord/src/adapters/postgresql/transaction-nested.test.ts`
(144 lines, mirroring
`vendor/rails/activerecord/test/cases/adapters/postgresql/transaction_nested_test.rb`)
is still adapter-level raw SQL on two `PostgreSQLAdapter` connections, and its
header documents a deliberate scope reduction: Rails' savepoint-scoped recovery
and final-state asserts (`[10, 10]` / `[2]` + `[1]`) are skipped because
"PostgreSQL dooms the whole serializable / deadlocked transaction on 40001 /
40P01". That premise needs re-checking against Rails now that the model layer is
in play — Rails asserts those exact final states on PG.

## Acceptance criteria

- [ ] `adapters/postgresql/transaction-nested.test.ts` rewritten through the
      model layer, following the shape now in
      `adapters/abstract-mysql-adapter/nested-deadlock.test.ts`
      (inline `Sample` model, `fixtures([], { useTransactionalTests: false })`,
      `withExecutionContext` for the second thread).
- [ ] Rails' final-state assertions restored, or the reduction re-justified
      against a fresh reading of `transaction_nested_test.rb` with a cited
      `file:line`.
- [ ] `parity:test` stays ✓ for the file; the tests still provoke the real
      serialization failure / deadlock. PostgreSQL lane green.
