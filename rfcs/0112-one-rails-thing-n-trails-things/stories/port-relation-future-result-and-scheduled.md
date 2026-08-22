---
title: "Relation keeps promise bookkeeping where Rails keeps @future_result/scheduled?"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: 40
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6530 wired `Relation#loadAsync` onto the FutureResult path, but kept
trails' promise-shaped bookkeeping instead of Rails' `@future_result`. The
relation surface Rails builds on that ivar is still unported
(`vendor/rails/activerecord/lib/active_record/relation.rb`):

- `load_async` (:1138-1152) runs inside `with_connection`, returns `load` when
  `!c.async_enabled?`, and stores the FutureResult in `@future_result` under
  `unless loaded?`. trails sets a private `_asyncLoad` flag and memoizes the
  in-flight promise in `_loadAsyncPromise` (`relation.ts`), computing
  `async_enabled?`/`joinable?` down in `_toArrayInner` because that is where the
  connection is in hand.
- `scheduled?` (:1169-1171) — `!!@future_result`. No trails counterpart, so
  nothing can ask whether a relation was scheduled.
- `then` (:1157-1165) — yields self through `@future_result.then` when
  scheduled, else `super`. trails' Relation is thenable through `toArray`, with
  no scheduled arm.
- `reset` (:1195-1196) — `@future_result&.cancel; @future_result = nil`. trails
  clears `_asyncLoad`/`_loadAsyncPromise` and never calls
  `FutureResult#cancel`, so a reset leaves the scheduled query running.
- `exec_queries` (:1403-1417) — reads `future.result` for the rows on the
  scheduled arm and nils `@future_result`. trails has no `exec_queries` /
  `exec_main_query` split at all; `_toArrayInner` is both.

The deviation is justified at the call site today (`relation.ts`, `loadAsync`)
on the grounds that trails awaits instantiation, eager loading and preloading
inside `_toArrayInner`, so the promise — not the FutureResult — is the handle a
second caller joins. That reasoning is a symptom of the missing
`exec_queries` / `exec_main_query` decomposition, not a language shortcoming:
Rails' future covers exactly the rows step because Rails has a separate
`instantiate_records` step (:1425-1433).

`FutureResult#cancel` being unreachable is the sharpest consequence — the
ported method (`future-result.ts`) has no production caller.

## Acceptance criteria

- `exec_queries` and `exec_main_query` exist as separate methods with Rails'
  names and bodies, `exec_main_query` taking `async:` and forwarding it to both
  its arms (relation.rb:1423-1441).
- `@future_result` is ported as a field: `load_async` stores it, `exec_queries`
  reads `future.result` and nils it, `reset` cancels it.
- `scheduled?` and the `then` scheduled arm are ported (relation.rb:1157-1171).
- `_asyncLoad` and `_loadAsyncPromise` are deleted, along with the call-site
  justification for them; concurrent `toArray()` callers still share one query.
- `FutureResult#cancel` has a production caller (`reset`), covered by a test
  asserting a reset relation's scheduled query is canceled.
- `pnpm parity:api:extra --package activerecord` does not grow; `scheduled?`
  and `then` land as matched Rails names.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
