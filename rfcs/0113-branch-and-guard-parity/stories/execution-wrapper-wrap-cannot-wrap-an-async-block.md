---
title: "ExecutionWrapper.wrap is sync-only, so async units of work hand-spell run!/complete!"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
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

Surfaced twice while writing PR #6532's tests.

`ActiveSupport::ExecutionWrapper.wrap`
(`vendor/rails/activesupport/lib/active_support/execution_wrapper.rb:120-137`)
takes a block, runs it inside the execution and calls `instance.complete!` in an
`ensure`:

```ruby
def self.wrap(source: "application.active_support")
  return yield if active?
  instance = run!
  begin
    yield
  rescue => error
    error_reporter.report(error, handled: false, source: source)
    raise
  ensure
    instance.complete!
  end
end
```

`packages/activesupport/src/execution-wrapper.ts`'s `wrap` is a faithful
synchronous port. But most trails work units are async, and a synchronous `wrap`
around an async block runs `completeBang()` at the first `await` — closing the
query-cache and the async-query session while the body is still running. Two
call sites had to spell the bracket out by hand as a result:

- `packages/activerecord/src/query-cache.test.ts`'s `middleware` helper, whose
  Rails counterpart (`query_cache_test.rb:841-845`) is one `executor.wrap`.
- `packages/trailties/src/application/executor-seam.trails.test.ts`.

`ActionDispatch::Executor` gets away with it because Rails itself splits the
bracket there — `run!` at the top of `call`, `state.complete!` deferred to the
Rack body's close (`actionpack/lib/action_dispatch/middleware/executor.rb:13-31`)
— which is exactly the shape both hand-written brackets copy. So the hand-spelling
is defensible, but it is a divergence from `wrap` that every async consumer will
hit, and there is no ported surface that wraps an async unit of work today.

Related: `packages/activerecord/src/query-cache.ts`'s `cache` / `uncached`
(`ClassMethods`) are already async and await their blocks, so the pattern for an
awaited block exists in the codebase.

## Acceptance criteria

1. Decide and document the settled trails shape for wrapping an _async_ unit of
   work in an execution — either an awaited overload of `wrap` that keeps the
   Rails name and control flow (`active?` early return, error report, `ensure
complete!`), or an explicit statement that `run!`/`complete!` is the async
   bracket and `wrap` is sync-only.
2. Whatever lands, `execution_wrapper.rb:120-137`'s branch order, the
   `source:` default and the error-report call are preserved.
3. `query-cache.test.ts`'s `middleware` helper collapses back toward its Rails
   one-liner (`query_cache_test.rb:841-845`) if the async shape allows it.
4. `pnpm parity:api:calls` / `parity:api:calls:args` stay green;
   `pnpm parity:api:extra --package activesupport` gains no untagged surface.
5. `executor.test.ts`, `reloader.test.ts`, `query-cache.test.ts` and
   `executor-seam.trails.test.ts` stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
