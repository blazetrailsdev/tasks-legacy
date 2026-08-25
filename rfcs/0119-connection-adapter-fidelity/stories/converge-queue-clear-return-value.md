---
title: "Queue#clear returns removed elements; Rails' Array#clear does not"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the `Queue` bodies in PR #6390 (RFC 0084).

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/connection_pool/queue.rb:50-54`:

```ruby
# Remove all elements from the queue.
def clear
  synchronize do
    @queue.clear
  end
end
```

`Array#clear` empties the array and returns the (now empty) array. The trails
port in
`packages/activerecord/src/connection-adapters/abstract/connection-pool/queue.ts`
(`Queue#clear`) instead snapshots the contents first and RETURNS THE REMOVED
ELEMENTS:

```ts
clear(): DatabaseAdapter[] {
  return synchronize(this, () => {
    const items = [...this._queue];
    this._queue = [];
    return items;
  });
}
```

The invented return value has callers. In
`packages/activerecord/src/connection-adapters/abstract/connection-pool.ts`,
`disconnect`/`discardConnections`/`clearReloadableConnections` call
`this._available?.clear()` for its side effect only, but one site
(around `const all = this._available.clear();`, in the reap/flush path) consumes
the returned array to decide which connections to re-add. Rails gets that list
from `@connections` / `@available` bookkeeping instead, never from `clear`'s
return.

## Acceptance criteria

- [ ] `Queue#clear` mirrors `queue.rb:50-54` — empties `@queue` and does not
      manufacture a list of removed elements for callers.
- [ ] Each caller in `connection-pool.ts` that consumed the return value is
      converged to the Rails source of that list; cite the Rails `file:line`
      for each at the call site.
- [ ] `pnpm parity:api:calls` / `:args` stay only-shrink; no new baseline rows.
