---
title: "preload_associations takes only records — drop trails' promoted-spec second argument"
status: draft
updated: 2026-08-16
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

Rails calls `preload_associations(records)` with one argument and derives the
spec list inside the method — `preload = preload_values; preload +=
includes_values unless eager_loading?`
(`vendor/rails/activerecord/lib/active_record/relation.rb:1321-1322`), called
from `exec_queries` (`:1414`).

trails passes a **second** argument: the specs not already promoted to an eager
JOIN. That is forced by the per-association `includes` promotion tracked in
[[includes-promotion-per-association-vs-eager-loading]] — Rails' `eager_loading?`
is all-or-nothing, so its callee can recompute the list; trails' caller knows
which subset was promoted and the callee does not.

PR #6604 surfaced this as a call-ARGUMENT baseline row once the live load path
took the `exec_queries` name:
`scripts/api-compare/call-mismatches-exclude/activerecord/relation.json`,
`kind: "args"`, `call: "preload_associations"`, `rubyArgs: ["ref:records"]`.

## Converged shape

`preloadAssociations(records)` — one argument, deriving the list from
`preloadValues` + `includesValues` internally, exactly as `relation.rb:1321-1322`
does. This unblocks only after the promotion divergence converges, so treat that
story as the dependency; this one retires the baseline row and the second
parameter.

## Acceptance criteria

- [ ] `preloadAssociations` takes `records` only; no caller passes a spec list.
- [ ] The `kind: "args"` `preload_associations` row is deleted from
      `call-mismatches-exclude/activerecord/relation.json` (only-shrink).
- [ ] `pnpm parity:api:calls:args` clean; `relations.test.ts` and
      `associations/eager.test.ts` pass unchanged on all three adapters.
