---
title: "ExecutionContext's store is a process-global Map, not per-execution-context via IsolatedExecutionState"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
packages: []
deps: []
deps-rfc: []
est-loc: 120
pr: null
claim: "2026-08-25T17:06:37Z"
assignee: "converge-no-touching-block-onto-apply-to"
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::ExecutionContext` (`vendor/rails/activesupport/lib/active_support/execution_context.rb`)
stores its context per-Fiber via `ActiveSupport::IsolatedExecutionState`
(`:8-10`, `:14-16`, `:24-26` — every accessor goes through
`IsolatedExecutionState[:active_support_execution_context]`), so two concurrent
requests never see each other's context.

trails' `packages/activesupport/src/execution-context.ts` backs the store with a
**process-global `Map`** (`const _store = new Map<string, unknown>()`, line 9),
and says so in its own header comment: "In production with concurrent async
requests, consider integrating with AsyncLocalStorage for isolation."

This was latent while nothing depended on it. PR #6302 made it load-bearing:
`ErrorReporter#set_context` now correctly forwards to
`ActiveSupport::ExecutionContext.set` (`error_reporter.rb:201-203`) and `report`
merges `ExecutionContext.to_h` (`:224`), so **error-report context now leaks
between concurrent tasks** — request A's `section: "admin"` can be attached to
request B's error report.

The machinery to fix it already exists in the same package and is already
async-scoped: `packages/activesupport/src/isolated-execution-state.ts` wraps
`getAsyncContext()` (AsyncLocalStorage) and exposes `get`/`set`/`fetch`/`run`.
That is precisely what Rails' `ExecutionContext` delegates to.

## Converged shape

Route `execution-context.ts`'s `_store` through `IsolatedExecutionState` under a
single key, as `execution_context.rb` does:

```ruby
def store
  IsolatedExecutionState[:active_support_execution_context] ||= {}
end
```

i.e. `IsolatedExecutionState.fetch("active_support_execution_context", () => new Map())`
in place of the module-level `Map`, leaving `set`/`setKey`/`toH`/`clear` and the
`afterChange` callbacks otherwise untouched. Note `IsolatedExecutionState` falls
back to a process-global Map when no scope is open, which preserves today's
top-level behaviour for callers outside a request.

## Acceptance criteria

- [ ] `ExecutionContext`'s store is per-execution-context, via
      `IsolatedExecutionState`, matching `execution_context.rb:8-26`.
- [ ] Context set inside `IsolatedExecutionState.run()` is invisible outside it
      and to a sibling `run()`.
- [ ] `ErrorReporter#report`'s `full_context` no longer leaks across concurrent
      tasks (`error_reporter.rb:224`).
- [ ] The "consider integrating with AsyncLocalStorage" note at the head of
      `execution-context.ts` is removed, not reworded.
- [ ] `error-reporter.test.ts` stays at 32/32 on `parity:test`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
