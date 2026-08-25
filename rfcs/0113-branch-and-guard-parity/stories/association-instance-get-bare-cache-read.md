---
title: "associationInstanceGet drops the hasCachedData gate for Rails' bare @association_cache read"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
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

Rails' `association_instance_get`
(`vendor/rails/activerecord/lib/active_record/associations.rb`) is a bare
`@association_cache[name]` read: any association that has been _built_ —
by a `person.pets` reader, by `record.association(:pets)` — is truthy for
every caller.

trails' `associationInstanceGet`
(`packages/activerecord/src/associations.ts`, the `hasCachedData` gate)
returns `null` unless the holder carries cached target data (a preloaded
holder, a loaded proxy, built records, or `isLoaded()`). A
built-but-unloaded association is therefore invisible to callers that
Rails answers.

That gate silently skipped Rails' second `reset_scope` caller:
`save_collection_association`
(`vendor/rails/activerecord/lib/active_record/autosave_association.rb:420-428`)
opens with `if association = association_instance_get(reflection.name)`
and then runs `association.reset_scope` at :428 — so in Rails an owner's
save always reconstructs the scope of any built association once its id
is known. PR #6609 had to compensate with a `!association` branch in
`packages/activerecord/src/autosave-association.ts` that resets the raw
`_associationInstances` entry directly. That branch is a stand-in for the
gate, not a Rails shape.

Regression cover for the symptom lives at
`packages/activerecord/src/associations/constructor-form-and-hmt-insert.test.ts`
(`resetScope on owner save > clears the scope of a built association that
never loaded a target`).

## Converged shape

Make `associationInstanceGet` the bare cache read Rails has, then delete
the compensating `!association` branch in `saveCollectionAssociation` so
the Rails body reads `if association = association_instance_get(...)` /
`association.reset_scope` with no trails-only arm.

The `hasCachedData` gate exists because trails splits the holder across
`_associationInstances` and `_collectionProxies` and some callers treat a
truthy-but-empty holder as "preloaded to nil" — audit those callers and
move the emptiness check to whichever of them actually needs it, rather
than to the shared accessor. Overlaps
[[homogenize-association-instances-cache]], which cleans up the same map
from the holder-shape side.

## Acceptance criteria

- [ ] `associationInstanceGet` returns the cached holder without the
      `hasCachedData` gate, matching Rails' `@association_cache[name]`.
- [ ] `saveCollectionAssociation`'s `!association` compensating branch is
      deleted; the body matches `autosave_association.rb:420-428`.
- [ ] `constructor-form-and-hmt-insert.test.ts`'s built-but-unloaded
      reset_scope cover stays green, along with the association and
      autosave suites, on SQLite, PostgreSQL and MySQL/MariaDB.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
