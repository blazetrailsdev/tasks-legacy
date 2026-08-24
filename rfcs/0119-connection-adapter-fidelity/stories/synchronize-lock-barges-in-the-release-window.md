---
title: "TransactionManager#synchronize is check-then-set, not a queue: a same-turn arrival overtakes a parked waiter"
status: draft
updated: 2026-08-10
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`TransactionManager#synchronize`
(`packages/activerecord/src/connection-adapters/abstract/transaction.ts`, the
`synchronize` method) is the port of Rails'
`@connection.lock.synchronize` (`activerecord/lib/active_record/connection_adapters/abstract/transaction.rb:622`
and the surrounding critical sections), and it is also the lock
`AbstractAdapter#withRawConnection` routes through
(`abstract_adapter.rb:972-984`). It is a check-then-set loop:

```ts
while (this._lockChain) await this._lockChain;
// ... then create a new _lockChain and claim ownership
```

That holds mutual exclusion — the re-check and the claim are synchronous, so
the first woken continuation wins — but it is **not a queue**. In the release
window, after `release()` nulls `_lockChain` and before an already-parked
waiter's continuation drains, a caller arriving in the same synchronous turn
observes a free lock and claims it AHEAD of the waiter.

Reproduced against the identical shape while converging the sqlite statement
lock in PR #6299: with a holder, one queued waiter `W`, and a late arrival `C`
entering in the same turn as the release, the check-then-set lock serves
`C,W` where a queue serves `W,C`. Ruby's `Monitor` is FIFO-fair per thread, so
Rails has no analogous barge.

Consequence: unbounded barging is a starvation hazard for a parked
`with_raw_connection` caller under sustained load, and it makes lock-order
reasoning depend on continuation scheduling rather than arrival.

Note the reentrancy here is deliberate and must be preserved — the lock is
shared between `withRawConnection` and every TransactionManager critical
section specifically to avoid an ABBA deadlock, and it re-enters per async
chain via the `_lockOwner` AsyncLocalStorage. This story is about the FIFO
discipline for the NON-reentrant path only.

## Converged shape

Replace the check-then-set loop with a tail-chain queue, the shape PR #6299
landed for the sqlite statement lock
(`connection-adapters/sqlite3/database-statements.ts`, `acquireStatementLock`):
each caller reads the current tail, publishes `tail.then(() => mine)` as the
new tail, and only then awaits the tail it found — all synchronously before its
first `await`, so arrival order fixes service order and there is no window in
which two callers both observe a free lock.

Keep the reentrant early-return (`storage.getStore() === this._currentLockOwner`)
exactly as it is; only the contended path changes.

Filed under 0023 because 0057-transaction-fidelity is closed; this is a
transaction-lock fidelity gap surfaced by PR #6299 rather than part of that
RFC's shipped scope.

## Acceptance criteria

- [ ] `synchronize` serves waiters in arrival order; a caller entering in the
      same synchronous turn as a release does not overtake a parked waiter.
- [ ] A regression test reproduces the barge and fails on the current
      implementation (holder + queued waiter + same-turn late arrival asserts
      `["queued", "late"]`).
- [ ] Reentrance is unchanged: a nested `synchronize` on the same async chain
      still returns without queueing, and the `withRawConnection` +
      `materializeTransactions` ABBA case stays deadlock-free.
- [ ] `transactions.test.ts` and the pool/concurrency suites stay green on all
      three adapters.
