---
title: "port-dependencies-interlock-permit-concurrent-loads"
status: draft
updated: 2026-08-12
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

Surfaced in review of PR #6390, which converged the `Queue#wait_poll` body to
Rails' loop shape.

`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/connection_pool/queue.rb:116-118`
wraps the blocking wait:

```ruby
ActiveSupport::Dependencies.interlock.permit_concurrent_loads do
  @cond.wait(timeout - elapsed)
end
```

The interlock is the autoload lock: a thread about to block on the pool
releases its share so another thread can autoload constants meanwhile, which
is what keeps a checkout-timeout from deadlocking against a load.

`packages/activerecord/src/connection-adapters/abstract/connection-pool/queue.ts`
(`Queue#waitPoll`) awaits `this._cond.wait(...)` directly, with a
`@missingRailsCall permit_concurrent_loads` tag at the call site. The reason is
that neither `ActiveSupport::Dependencies.interlock` nor Zeitwerk-style
autoloading is ported at all — see the BLOCKED note in
`packages/actionpack/src/action-dispatch/dispatch/debug-locks.test.ts:9-15`,
which scopes the interlock port at ~80 LOC and blocks `DebugLocks` on the same
gap.

So this story is downstream of that port: there is no interlock to permit
concurrent loads on until `ActiveSupport::Dependencies.interlock` exists.

## Acceptance criteria

- [ ] Decide whether `ActiveSupport::Dependencies.interlock` is portable to
      trails at all (Node has no autoload interlock; the `debug-locks` note is
      the prior art). If it is not, close this by recording that finding —
      with the Rails cite — and converting the `@missingRailsCall` tag's text
      to point at the decision rather than at this story.
- [ ] If it is portable, `Queue#waitPoll` wraps its `await this._cond.wait(...)`
      in the ported `permitConcurrentLoads`, and the `@missingRailsCall` tag on
      `waitPoll` is deleted.
- [ ] `packages/actionpack/.../debug-locks.test.ts`'s BLOCKED note is revisited
      in the same pass, since it names the same missing primitive.
