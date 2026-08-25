---
title: "PG transactionStatus synthesizes PQTRANS_ACTIVE from a hand-maintained flag"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails reads libpq's real connection state:

```ruby
# postgresql/database_statements.rb:127-133
IDLE_TRANSACTION_STATUSES = [PG::PQTRANS_IDLE, PG::PQTRANS_INTRANS, PG::PQTRANS_INERROR]

def cancel_any_running_query
  return if @raw_connection.nil? || IDLE_TRANSACTION_STATUSES.include?(@raw_connection.transaction_status)
  @raw_connection.cancel
  @raw_connection.block
end
```

`PQtransactionStatus` is computed by libpq from the protocol stream. trails'
`get transactionStatus`
(`packages/activerecord/src/connection-adapters/postgresql-adapter.ts`, the
`PQtransactionStatus` port) instead synthesizes the PQTRANS_ACTIVE arm from two
JS-side flags:

```ts
if (client.readyForQuery !== true && !this._commandSettled) return PQTRANS_ACTIVE;
```

`_commandSettled` is hand-maintained: ~10 sites set it `false` before putting a
command on the wire, and `_attachReadyForQueryListener` sets it `true` again
from the driver's `readyForQuery` / `commandComplete` / `errorMessage` events.
Any site that flips it `false` without a matching completion on the SAME
`pg.Client` strands the adapter reporting PQTRANS_ACTIVE forever — so
`cancel_any_running_query`'s gate stops returning early and every rollback fires
a CancelRequest at a backend that is in fact idle-in-transaction. A
CancelRequest is addressed to a backend, not a statement, so it lands on
whatever query runs next.

That is exactly the defect PR #6606 fixed one instance of: the statement pool's
eviction DEALLOCATEs ran outside the connection lock, so the flag was `false`
with nothing this chain owned in flight. Measured on a full PG shard-2 run —
every production cancel reported `readyForQuery=false settled=false
rfqStatus=T`, i.e. a ReadyForQuery with status 'T' (idle-in-transaction) had
already been recorded. #6606 removed that one unsettling site; the invariant is
still maintained by hand at every other one.

## Acceptance criteria

- `transactionStatus` derives the PQTRANS_ACTIVE arm from driver state rather
  than a hand-maintained per-adapter flag, so no call site can strand it.
  node-pg's `Client#readyForQuery` and `activeQuery` are the candidates; a
  `_commandSettled` that only ever mirrors those is not a fix.
- If a flag genuinely cannot be eliminated, its false-setting sites must be
  paired with their completion structurally (one helper that owns both edges),
  not by convention at ~10 call sites.
- The four status arms still map to `PQTRANS_IDLE` / `INTRANS` / `INERROR` /
  `ACTIVE` as `PQtransactionStatus` does, and `IDLE_TRANSACTION_STATUSES`
  (`postgresql/database_statements.rb:128`) keeps its Rails membership.
- Regression cover that a stranded/unpaired command cannot make a rollback fire
  a CancelRequest at an idle-in-transaction backend.
- Related, do not duplicate: `converge-clear-cache-lock-mysql-sqlite`
  (the same missing `@lock.synchronize`, other adapters) and
  `pg-reset-bang-rollback-gate-uses-client-not-transaction-status`
  (a different gate on the same getter).
