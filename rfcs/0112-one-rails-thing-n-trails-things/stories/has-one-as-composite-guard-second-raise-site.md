---
title: "Converge or remove the trailing CompositePrimaryKeyMismatchError raises in _findHasOneTarget"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
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

PR #5925 removed the singular no-reflection load fallback and hoisted a
`AssociationNotFoundError` guard to the top of `_findBelongsToTarget` and
`_findHasOneTarget`
(`packages/activerecord/src/associations/singular-association.ts`), mirroring
Rails' raise in `association` (`associations.rb:51-56`) before
`Association#initialize` (`association.rb:38-45`) ever constructs from a
reflection.

That leaves the two polymorphic `:as` composite-key guards in
`_findHasOneTarget` (roughly the `if (options.as)` block: array-`foreignKey`, and
composite `primaryKey` without `"id"`) in a state the earlier story
`composite-pk-mismatch-extra-guard-raise-sites` could not resolve. Each calls
`routeThroughCheckValidity(ctor, assocName)` — Rails' single raise site,
`reflection.check_validity!` — and then throws a second, trails-invented
`CompositePrimaryKeyMismatchError` if that call returns. Before #5925 the
trailing throw was the "no reflection resolvable" escape; now the reflection is
guaranteed non-null by the guard above it, so the throw is reachable only when
`check_validity!` declines to raise for a shape Rails accepts — i.e. it may be
strictly-divergent dead code.

Rails has no counterpart raise inside `find_target`: `HasOneAssociation`
inherits `SingularAssociation#find_target` (`singular_association.rb:47-55`),
which builds `scope` and loads; every composite-key validity complaint comes
from `check_validity!` at construction.

## Acceptance criteria

- Determine, by instrumentation over the AR suites, whether either trailing
  `CompositePrimaryKeyMismatchError` throw in `_findHasOneTarget` is reachable
  now that a validated reflection precedes it.
- If unreachable, delete both throws and keep only the
  `routeThroughCheckValidity` call (Rails' single raise site).
- If reachable, cite the Rails shape that reaches it and converge the raise onto
  `check_validity!` rather than leaving a second raise site in the loader.
- `associations/singular-association.ts` stays at 0 novel extra surface.
- Association suites pass with no test renames.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
