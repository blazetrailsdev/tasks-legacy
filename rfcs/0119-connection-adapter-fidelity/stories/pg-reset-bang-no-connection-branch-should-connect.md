---
title: "PG resetBang's no-connection branch runs super where Rails returns connect!"
status: draft
updated: 2026-08-11
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `PostgreSQLAdapter#reset!`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql_adapter.rb:371-381`)
opens with:

```ruby
def reset!
  @lock.synchronize do
    return connect! unless @raw_connection
    ...
```

so with no raw connection the reset _reconnects_ (`connect!` is `verify!`,
`abstract_adapter.rb:778-781`) and never reaches `super`.

trails' `resetBang` (`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`)
instead does the opposite in that branch:

```ts
if (!this._rawConnection) {
  super.resetBang();
  return;
}
```

— it runs `super` (the branch Rails skips) and never connects. A pool checkin
on a torn-down adapter therefore leaves it unconnected where Rails would have
re-established and verified the connection; the gap is only masked because the
next checkout's `verifyBang` reconnects.

Pre-existing; surfaced during review of #6376 (RFC 0061
`pg-reset-body-under-one-lock`), which left the branch untouched.

## Acceptance criteria

- The no-connection branch calls `connectBang()` (trails' `connect!`) and
  returns, inside the lock, instead of dispatching to `super.resetBang()`.
- `resetBang` stays sync per `AbstractAdapter`; the connect hop is scheduled on
  the same lock the rest of the body uses (the shape #6376 established).
- A regression test pins that `resetBang()` on an adapter with no raw
  connection re-establishes it, and fails on the current baseline.
- PG lane green; `parity:api:calls` for `reset!` unchanged or improved.

## Absorbed: `pg-reset-bang-rollback-gate-uses-client-not-transaction-status`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "resetBang gates ROLLBACK on \_client, Rails gates on transaction_status"

### Context

`PostgreSQLAdapter#resetBang`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts:2783-2791`)
gates its ROLLBACK on `if (this._client)` — trails' pinned-client field.

Rails gates on the connection's transaction status:

```ruby
# postgresql_adapter.rb:371-381
def reset!
  @lock.synchronize do
    return connect! unless @raw_connection

    unless @raw_connection.transaction_status == ::PG::PQTRANS_IDLE
      @raw_connection.query "ROLLBACK"
    end
    @raw_connection.query "DISCARD ALL"

    super
  end
end
```

The adapter already ports `transaction_status` — `get transactionStatus`
(`postgresql-adapter.ts:1981`, mirroring `PQtransactionStatus`) — and
`IDLE_TRANSACTION_STATUSES` is already used by `_cancelAnyRunningQuery`
(`postgresql/database_statements.rb:128`), so the Rails-shaped condition is
available at the call site with no new machinery.

The two are not equivalent: `_client` is non-null only when trails pinned a
client for an explicitly-begun transaction, whereas `PQTRANS_INTRANS` /
`PQTRANS_INERROR` also cover a transaction the server opened that trails did not
pin. Where they differ, trails skips a ROLLBACK Rails would send, leaving
DISCARD ALL to run inside an open transaction block.

Found while reviewing `resetBang` for PR #6365 (which removed a non-Rails
CancelRequest from the same method).

### Acceptance criteria

- `resetBang` gates the ROLLBACK on the transaction status, as
  `postgresql_adapter.rb:375-377` does, not on `_client`.
- The `_client = null` / `_inTransaction = false` bookkeeping keeps whatever
  gate it needs — it tracks trails' pinned client, which Rails has no analogue
  for, and is a separate concern from whether ROLLBACK is sent.
- A test covers the divergent case: a server-side transaction the adapter did
  not pin is rolled back by `resetBang`.

### Definition of done

PG suites green (`adapters/postgresql/**`, `connection-adapters/**`,
`adapter.test.ts`, `transactions.test.ts`) against PG 17.

### Verification

`pnpm parity:api:calls` — check whether the row for `reset!` moves.
