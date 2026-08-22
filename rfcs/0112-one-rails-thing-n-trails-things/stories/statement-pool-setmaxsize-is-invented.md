---
title: "StatementPool#setMaxSize has no Rails counterpart — the limit is constructor-only"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
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

Rails' `StatementPool` sets its bound once, in the constructor, and never
exposes a mutator:

```ruby
def initialize(statement_limit = nil)
  @cache = Hash.new { |h, pid| h[pid] = {} }
  @statement_limit = statement_limit || DEFAULT_STATEMENT_LIMIT
end
```

(`vendor/rails/activerecord/lib/active_record/connection_adapters/statement_pool.rb:10-13`.)
There is no `statement_limit=`, no resize, and no eviction-on-resize path —
the only eviction is the `while @statement_limit <= cache.size` loop inside
`[]=` (`:31-33`).

trails' `StatementPool` adds `setMaxSize(maxSize)`
(`connection-adapters/statement-pool.ts:31`), which validates the argument,
mutates `_maxSize`, and runs its own eviction loop with a `dealloc` sweep. It
is public, novel surface — `pnpm parity:api:extra --package activerecord` reports
`connection-adapters/statement-pool.ts — 3 novel`, and this is one of them. A
`maxSize` getter reads it back.

Surfaced by `should-prepare-statement-limit-gate-is-invented` (PR #6293): the
now-deleted `_shouldPrepare` limit gate cited a direct `pool.setMaxSize(0)`
"by a test or an operator shrinking the session" as its justification, and the
only remaining callers are trails-only tests.

## Converged shape

Delete `setMaxSize` (and the `maxSize` getter if it has no Rails reader), so
the limit is constructor-only as in `statement_pool.rb:10-13`. The trails-only
tests that resize a live pool should construct a pool at the limit under test
instead — the zero-limit test in `statement-pool.test.ts` already has a
constructor-based sibling proving that path.

## Acceptance criteria

- [ ] `setMaxSize` is gone; the limit is set only at construction.
- [ ] Tests that resized a live pool construct one at the target limit.
- [ ] `pnpm parity:api:extra --package activerecord` loses the row(s).
- [ ] parity:api / parity:test delta non-negative; all three lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
