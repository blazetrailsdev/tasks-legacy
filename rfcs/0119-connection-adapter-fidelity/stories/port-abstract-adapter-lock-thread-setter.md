---
title: "Port AbstractAdapter#lock_thread= and free the lockThread name"
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

Rails installs the connection's monitor in a setter, choosing the lock by what
owns the connection:

```ruby
def lock_thread=(lock_thread) # :nodoc:
  @lock =
  case lock_thread
  when Thread then ActiveSupport::Concurrency::ThreadLoadInterlockAwareMonitor.new
  when Fiber  then ActiveSupport::Concurrency::LoadInterlockAwareMonitor.new
  else             ActiveSupport::Concurrency::NullLock
  end
end
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:181-192`.)

PR #6424 gave `AbstractAdapter` a real `lock` — a
`LoadInterlockAwareMonitor` built in the field initializer
(`packages/activerecord/src/connection-adapters/abstract-adapter.ts`) — but did
NOT port the setter. `lockThread` exists on the adapter as an unrelated plain
`boolean` field (abstract-adapter.ts, `lockThread: boolean = false`), so the
Rails name is currently taken by something that is not the Rails member, and
`NullLock` (`packages/activesupport/src/concurrency/null-lock.ts`) has no
installer — nothing can ever select it.

Rails' caller is `ConnectionPool#pin_connection!` / `unpin_connection!`
(`connection_pool.rb`), which sets `lock_thread` to pin a connection to the
owning thread/fiber for transactional fixtures; trails' `pinConnectionBang`
takes a `_lockThread` argument it currently ignores for lock purposes
(`packages/activerecord/src/connection-adapters/abstract/connection-pool.ts:732`).

## Converged shape

`setLockThread(lockThread)` (the settled trails spelling for a Ruby `x=` that
cannot be a TS `set` accessor — see CLAUDE.md) assigns `this.lock` from the
argument, with `NullLock` as the `else` arm. trails has one concurrency model,
so the `Thread`/`Fiber` split collapses: decide whether the truthy arm is always
`LoadInterlockAwareMonitor` (and say so at the call site with the Rails cite) or
whether `ThreadLoadInterlockAwareMonitor` needs a port at all. Then wire
`pinConnectionBang`/`unpinConnectionBang` to it so the pinned-connection path
selects the lock the way `connection_pool.rb` does, and resolve the
`lockThread: boolean` field name collision.

## Acceptance criteria

- [ ] The setter exists at Rails' name and assigns `@lock`, matching
      abstract_adapter.rb:181-192, including the `NullLock` arm.
- [ ] The `lockThread: boolean` field no longer squats on the Rails name.
- [ ] `pinConnectionBang` drives it, matching Rails' pin/unpin.
- [ ] Transactional-fixture and connection-pool suites green on all three lanes.
