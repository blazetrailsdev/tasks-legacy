---
title: "Open async-queries sessions from an executor boundary, not a lazy seed"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails opens and closes an asynchronous-queries **session** per request from the
Executor: `ActiveRecord::AsynchronousQueriesTracker.run` / `.complete`
(`vendor/rails/activerecord/lib/active_record/asynchronous_queries_tracker.rb:32-40`),
installed as executor hooks by the railtie. `async_query_executor` work then
schedules its `FutureResult` onto `asynchronous_queries_session`, which exists
only inside such a session.

trails has no Executor to hook. `asynchronousQueriesTracker()`
(`packages/activerecord/src/core.ts:655-659`) instead lazily seeds a tracker
through `IsolatedExecutionState` as a stopgap, and
`asynchronousQueriesSession()` (core.ts:667-669) reads `currentSession` off it.

Two consequences, both visible today:

- The seeded tracker is shared through the isolated execution state, so a
  sibling suite's `complete()` can empty the stack out from under an unrelated
  caller. `packages/activerecord/src/relation-load-async.trails.test.ts:32-44`
  documents exactly this and works around it by opening its own session in
  `beforeEach`, "exactly as the Executor would".
- Nothing ever calls `complete()` on the request boundary, so sessions are not
  scoped the way Rails scopes them.

The comment at relation-load-async.trails.test.ts:40 cites story
`install-executor-hooks-for-async-queries-tracker`, which is `status: done`
(PR 6529) — the citation is stale, but the deviation it describes is still
present in `core.ts`. Discovered while migrating call-set baseline reasons in
PR #6850.

## Converged shape

An executor/reloader analogue that brackets a unit of work with
`AsynchronousQueriesTracker.run()` / `.complete()`, so `asynchronousQueriesSession()`
resolves inside a real session and the lazy `IsolatedExecutionState` seed in
`core.ts:655-659` can be deleted rather than relied on.

## Acceptance criteria

- [ ] A session is opened and completed at a request/work boundary, mirroring asynchronous_queries_tracker.rb:32-40, rather than seeded lazily on first read.
- [ ] `asynchronousQueriesTracker()`'s stopgap seeding in core.ts:655-659 is deleted.
- [ ] `relation-load-async.trails.test.ts`'s `beforeEach` no longer has to stand in for the Executor, and its explanatory comment (and the stale story citation at line 40) goes with it.
- [ ] All three DB lanes green.
