---
title: "Converge has_one :through off the two-step load onto the scope chain"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by #5911, which collapsed the singular loader dispatcher into one
Rails-shaped `findTarget`
(`packages/activerecord/src/associations/singular-association.ts`). After that
collapse, `has_one :through` routing is the only genuinely macro-conditional
step left in the body that is not a cache read:

```ts
if (!isBelongsTo && options.through) {
  if (_canRouteThroughViaDisableJoinsAssociationScope(reflection, options)) {
    return _loadSingularThroughViaDisableJoinsScope(record, reflection, options);
  }
  if (!_routeThroughViaAssociationScope(record, reflection, options)) {
    const { findTarget: findThroughTarget } = await import("./has-one-through-association.js");
    return findThroughTarget(record, assocName, options);
  }
  // Otherwise fall through to the scope path below.
}
```

Rails has no such branch. `SingularAssociation#find_target`
(`vendor/rails/activerecord/lib/active_record/associations/singular_association.rb:47-55`)
treats `:through` like any other association because `ThroughAssociation#scope`
(`through_association.rb`) folds the through chain into the scope the reflection
builds, and `HasOneThroughAssociation`
(`has_one_through_association.rb`) defines no `find_target` at all — it overrides
only the replace/persistence side. The one conditional Rails does have here is
`disable_joins`, which is exactly the first branch above.

So the deviation is the second branch: the shapes
`_routeThroughViaAssociationScope` rejects fall back to a **two-step load**
(load the through record, then load the source off it) in
`has-one-through-association.ts`, where Rails issues one scope-built query. That
also names a `find_target` in a file where Rails declares none.

The analogous nested-through routing gate for collections was converged by
story `converge-nested-through-routing-to-association-scope` (RFC 0023, PR 4505),
which widened AssociationScope's routing until the bail was unnecessary. This is
the singular counterpart of that work.

## Acceptance criteria

- Enumerate the shapes `_routeThroughViaAssociationScope` currently rejects, and
  for each, either widen the AssociationScope path to build it or record why the
  chain genuinely cannot express it.
- Remove the two-step fallback branch from the unified `findTarget`, leaving
  `disable_joins` as the only `:through` conditional — matching
  `singular_association.rb:47-55`.
- `has-one-through-association.ts` no longer exports a `find_target` counterpart
  Rails does not declare there (see `has_one_through_association.rb`).
- No new wide call-mismatch baseline entries; `singular-association.ts` and
  `has-one-through-association.ts` stay at 0 novel extra surface.
- has_one :through / nested-through / preloader suites pass on SQLite,
  PostgreSQL and MySQL with no test renames.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
