---
title: "Delete _findTargetReachable and gate the strict-loading raise on the real find_target? path"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 220
pr: null
claim: "2026-08-25T16:34:34Z"
assignee: "converge-association-relation-through-scope-onto-scoping"
blocked-by: null
closed-reason: null
---

## Context

Rails never guards the strict-loading raise itself. `find_target` raises
unconditionally as its first statement
(vendor/rails/activerecord/lib/active_record/associations/association.rb:248-250);
what keeps a non-querying read from raising is that `load_target` only calls
`find_target` when `find_target?` says a query is needed
(association.rb:190 and :320-321, overridden at
belongs_to_association.rb:124-126).

trails has no such gate on the loader paths, so it stands in a trails-only
predicate `_findTargetReachable(record, assocName, options, kind)`
(associations.ts:1420) that re-derives `find_target?` / `foreign_key_present?`
from the owner + options triple, and ANDs it with the raise at both loader
sites (associations/has-many-association.ts, associations/singular-association.ts).
It is a second implementation of a predicate the OO associations already have
as `findTargetNeeded()`.

Surfaced by PR #6472, which wired `violates_strict_loading?` into these sites
and left the `_findTargetReachable` conjunct in place.

## Acceptance criteria

1. The loaders reach the raise only through the real `find_target?` gate — the
   OO `findTargetNeeded()` / `Association#loadTarget` — so the raise site is
   Rails' unconditional `if violates_strict_loading?`.
2. `_findTargetReachable` is deleted from associations.ts along with its
   `"belongsTo"` / `"foreign"` kind discriminator.
3. `strict-loading.trails.test.ts` — which exists specifically to pin the
   `find_target?` gate for new-record owners, both the raising and the silent
   arms — stays green without modification.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
