---
title: "Retire Queue's invented rejectAll and leaseTo/unlease/leasedTo surface"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the `Queue` bodies in PR #6390 (RFC 0084).

`pnpm parity:api:extra --package activerecord` reports 4 novel names for
`packages/activerecord/src/connection-adapters/abstract/connection-pool/queue.ts`
with no counterpart in
`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/connection_pool/queue.rb`
(the whole file is 190 lines — the Rails surface is small and fully enumerable):

- `Queue#rejectAll(error)` — no Rails analogue. Rails never rejects waiters;
  a waiter either gets a connection or times out in `wait_poll`
  (`queue.rb:111-132`). Called from `connection-pool.ts` in the `disconnect!`
  and `discard!` paths, where Rails instead relies on the pool's own
  bookkeeping and the waiter's timeout.
- `ConnectionLeasingQueue#leaseTo` / `#unlease` / `#leasedTo` — a
  `Map<DatabaseAdapter, string>` of lease owners living on the queue. Rails'
  `ConnectionLeasingQueue` (`queue.rb:177-190`) has exactly one member,
  `internal_poll`, which calls `conn.lease` — the lease state lives on the
  CONNECTION (`abstract_adapter.rb#lease`), not in a side table on the queue.

(`length` is reported as "moved" rather than novel and is out of scope here.)

PR #6390 already removed two other invented names from this file
(`Queue#remove(conn)`, `waitingCount`) by converging their callers, taking the
count from 6 to 4.

## Acceptance criteria

- [ ] `leaseTo` / `unlease` / `leasedTo` are deleted; lease ownership is read
      from the connection as `queue.rb:186-188` does, or their callers are
      converged to whatever Rails actually uses.
- [ ] `rejectAll` is deleted and its `disconnect!` / `discard!` callers
      converged to Rails' shape, OR it is justified at the call site as a
      genuine TypeScript/Node shortcoming with the Rails cite (Node cannot
      leave a promise pending forever the way a blocked Ruby thread is simply
      killed with the pool) and carries a `@noRailsEquivalent` tag.
- [ ] `pnpm parity:api:extra --package activerecord` reports fewer novel names
      for this file than the 4 it reports today.
