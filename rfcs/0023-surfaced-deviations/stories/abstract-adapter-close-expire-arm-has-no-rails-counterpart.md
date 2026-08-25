---
title: "Converge AbstractAdapter#close onto Rails' bare pool.checkin"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while retiring `poolAbsent` / `realPool` in #5885.

Rails' `AbstractAdapter#close` is exactly one line
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:830`):

```ruby
def close
  pool.checkin self
end
```

Every adapter always carries a pool — `NullPool.new` by default
(`abstract_adapter.rb:153`) — and `NullPool#checkin` is a no-op
(`connection_pool.rb:39`), so the one line is total.

trails' `close` (`packages/activerecord/src/connection-adapters/abstract-adapter.ts`,
the `close()` method) instead branches:

```ts
const pool = this.pool as { checkin?: (conn: unknown) => void } | null;
if (pool && !(pool instanceof NullPool) && typeof pool.checkin === "function") {
  pool.checkin(this);
} else if (this._inUse) {
  this.expire();
}
```

The `else if (this._inUse) this.expire()` arm has no Rails counterpart. #5885
replaced the `poolAbsent(pool)` call with an inline `instanceof NullPool` test
but deliberately left the deviation itself intact — retiring the exported
helpers was the claimed scope.

Two sub-questions to answer against Rails:

1. Is the `typeof pool.checkin === "function"` duck-check needed at all, or is
   it defending against a pool shape that no longer exists?
2. Does anything actually depend on the `expire()` arm, or does
   `NullPool#checkin` being a no-op make it dead? `expire()` on a
   standalone adapter has no Rails analogue in this path.

## Acceptance criteria

- Determine, against `abstract_adapter.rb:830` and `connection_pool.rb:14-50`,
  whether `close()` can converge to a bare `this.pool.checkin(this)`.
- If yes: converge it, and drop the `_inUse` / `expire()` arm and the
  duck-check.
- If no: record the real reason at the method (replacing the current comment)
  — a negative result is valid as long as the reason is the true one.
- Existing tests pass; no test renames.
