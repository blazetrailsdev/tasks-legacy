---
title: "Nested-attributes collection arm still only marks already-loaded records"
status: draft
updated: 2026-07-30
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

`assignNestedAttributesForCollectionAssociation`
(`packages/activerecord/src/nested-attributes.ts:998-1010` on main) still carries
a "KNOWN LIMITATION vs Rails" comment: Rails computes `existing_records` as
`association.loaded? ? target : scope.where(pk => ids)`
(`vendor/rails/activerecord/lib/active_record/nested_attributes.rb:510-515`) —
i.e. it queries the DB when the collection is not loaded — while trails only
marks records that are already in memory. An unloaded collection is therefore
neither validated against nor marked for destruction at assignment; the rows are
only reached by the post-save flush.

This is the exact collection sibling of the one-to-one gap closed by PR #5643.
The blocker cited in the comment ("the sync setter can't perform trails' async
load") no longer holds: the one-to-one arm now issues the load at assignment and
parks the continuation on the owner-side drain
(`parkDisplacedRemoval` / `awaitPendingDisplacedRemovals`), which both `save()`
and the awaitable `set#{Name}Attributes` writer run. The same mechanism can carry
the collection arm.

A prior story, `nested-attr-collection-existing-records-query-when-unloaded`, is
marked `done`, but the limitation and its comment survive in merged main — so
that story either converged something adjacent or closed early. Confirm what it
actually shipped before starting, and fold this into it if they overlap.

## Acceptance criteria

- [ ] An unloaded collection's `id`-addressed nested attributes resolve at
      assignment (Rails' `scope.where(pk => ids)`), rather than only marking
      already-loaded records.
- [ ] `_destroy` on an unloaded collection member participates in pre-save
      validations against the post-destroy graph (e.g. the association-aware
      length validator), matching Rails.
- [ ] The "KNOWN LIMITATION vs Rails" comment is removed once converged.
- [ ] Regression test verified failing on the pre-fix baseline.
