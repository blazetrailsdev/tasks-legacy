---
title: "Reloader's unload lock omits Dependencies.interlock start_unloading/done_unloading"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-order
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

Surfaced in PR #6532, which ported `ActiveSupport::Reloader < ExecutionWrapper`.

`vendor/rails/activesupport/lib/active_support/reloader.rb:101-115` guards the
class unload with the autoload interlock:

```ruby
def require_unload_lock!
  unless @locked
    ActiveSupport::Dependencies.interlock.start_unloading
    @locked = true
  end
end

def release_unload_lock!
  if @locked
    @locked = false
    ActiveSupport::Dependencies.interlock.done_unloading
  end
end
```

`packages/activesupport/src/reloader.ts` ports the `@locked` bookkeeping and the
branch structure exactly, but makes neither interlock call — the omission is
cited in prose in both methods' JSDoc. It could not carry the sanctioned
`@missingRailsCall` tag: `Dependencies.interlock.*` is not in the call-set
ratchet's flagged population for this file, so the tag reds
`pnpm parity:api:calls` as a STALE justification (verified in both the
fully-qualified and bare `interlock` spellings).

`ActiveSupport::Dependencies::Interlock`
(`vendor/rails/activesupport/lib/active_support/dependencies/interlock.rb`) is
unported. The sibling call `permit_concurrent_loads` is tracked separately by
[[port-dependencies-interlock-permit-concurrent-loads]] (connection pool
`Queue#wait_poll`); this story is the Reloader half.

Note the stated reason for the omission is that ESM has no autoload, so there is
nothing to hold loads off across an unload — the same reason
`Concurrency::LoadInterlockAwareMonitor` is a bare `Monitor`
(`packages/activesupport/src/concurrency/load-interlock-aware-monitor.ts:7-18`).
That may make this a ratify-and-skip rather than a converge; if so it should be
settled once, in `SKIP_GROUPS` with a reason, rather than left as prose in two
method bodies.

## Acceptance criteria

1. Either port `ActiveSupport::Dependencies::Interlock`'s `start_unloading` /
   `done_unloading` and call them from `requireUnloadLockBang` /
   `releaseUnloadLockBang` per `reloader.rb:104`/`:113`, or record the
   no-autoload rationale once in `SKIP_GROUPS` in `scripts/parity/conventions.ts`
   and drop the duplicated prose from both JSDoc blocks.
2. Whichever way it lands, `pnpm parity:api:calls` stays green — do not
   reintroduce a `@missingRailsCall` tag for a call the ratchet does not flag.
3. `packages/activesupport/src/reloader.test.ts` stays green; if the interlock is
   ported, enroll whatever of `reloader_test.rb` covers the unload lock.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
