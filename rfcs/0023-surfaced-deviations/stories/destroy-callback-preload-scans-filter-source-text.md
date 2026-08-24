---
title: "Destroy-callback belongs_to preload reads callback source text; Rails loads lazily at the dereference"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/base.ts` carries a module-private
`beforeOrAroundCallbackSources(proto, event)` tagged
`@noRailsEquivalent CONVERGEABLE`, plus its two private siblings
`referencesAssociationName` and `expandCallbackSourcesWithHelpers`. Together
they read the **source text** of every registered `before_destroy` /
`around_destroy` filter via `Function.prototype.toString()`, regex-match
association names out of it, and use that to decide which `belongs_to` targets
to `loadTarget()` before running the destroy callbacks
(`_preloadBelongsToForDestroyCallbacks`).

Rails does none of this. A callback body dereferences an association and
`ActiveRecord::Associations::Association#load_target`
(`activerecord/lib/active_record/associations/association.rb:168-176`) runs at
that moment, lazily, through ordinary method dispatch — see
`ActiveRecord::Callbacks` (`activerecord/lib/active_record/callbacks.rb:326-359`)
and `Persistence#destroy` (`activerecord/lib/active_record/persistence.rb:719-731`),
neither of which inspects callback bodies. The scan exists because trails'
association reads are async while the callback chain has synchronous callers, so
the target has to be resolved _before_ the body runs.

This is heuristic and fails open in both directions: a filter that is an object
or method-name filter, a bound function, or a native function is `opaque` and
forces loading every `belongs_to`; a filter that reaches an association through
a helper beyond the one expansion level `expandCallbackSourcesWithHelpers`
performs is missed entirely. It was introduced to undo the per-association
savepoint churn PR #4792 paid.

This story was surfaced while retiring activemodel's callbacks adapter
(PR #6964). The scan moved from `packages/activemodel/src/callbacks.ts` into
`base.ts` there, so it is no longer on activemodel's public API, but the
deviation itself is untouched.

## Converged shape

No source-text introspection anywhere. A destroy callback that reads
`record.firm` resolves the target at the point of the read, as Rails does. The
likely route is making the association read path awaitable from inside a
callback body — i.e. resolving the same async-read-in-a-sync-chain problem that
`project_sync_wrappers_drop_promises_from_async_validators` and RFC 0063 (which
already made `isValid()` return `Promise<boolean>`) address elsewhere — so the
preload disappears rather than getting a better heuristic.

## Acceptance criteria

- `beforeOrAroundCallbackSources`, `referencesAssociationName`,
  `expandCallbackSourcesWithHelpers` and `_preloadBelongsToForDestroyCallbacks`
  are deleted from `packages/activerecord/src/base.ts`; no `toString()` of a
  callback filter remains in the repo.
- `belongs_to` targets named by a destroy callback are still loaded, and ones it
  never names are still not queried — verified by the existing destroy-callback
  tests plus a case whose association read is reached through two levels of
  helper method, which the current one-level expansion misses.
- No new savepoint-per-association churn (the regression PR #4792 paid and this
  scan was added to undo).
- `pnpm parity:api:extra --package activerecord` shows `base.ts` down by the
  three private helpers; no `@noRailsEquivalent` remains for this cluster.
