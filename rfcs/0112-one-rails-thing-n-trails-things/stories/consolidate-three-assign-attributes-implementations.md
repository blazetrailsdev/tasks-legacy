---
title: "Consolidate the three divergent _assign_attributes implementations"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 350
pr: null
claim: "2026-08-25T14:26:33Z"
assignee: "consolidate-three-assign-attributes-implementations"
blocked-by: null
closed-reason: null
---

## Context

Rails has exactly one `_assign_attributes`
(`vendor/rails/activerecord/lib/active_record/attribute_assignment.rb:6-22`),
reached from ActiveModel's `assign_attributes`
(`vendor/rails/activemodel/lib/active_model/attribute_assignment.rb:32-35`) and
therefore from `#new`, `#assign_attributes`, and `#update` alike.

trails has three divergent implementations of that one method:

1. `packages/activerecord/src/attribute-assignment.ts:32` `_assignAttributes` —
   the Rails-named private layer; dispatches via `this._assignAttribute`.
2. `packages/activerecord/src/persistence.ts` `assignAttributes` — the live
   `assign_attributes` / `#update` path; dispatches via a module-local
   `_assignAttribute` (prototype setter → `writeAttribute`, wrapped in
   `AttributeAssignmentError`). Ported the nested deferral in PR #6003.
3. `packages/activerecord/src/base.ts:3060` constructor — a bespoke
   `hasMultiparameterKeys` / `_withoutDeferredConstructionKeys` split with no
   nested bucket at all.

They disagree on the Hash test, on whether nested hashes are deferred, and on
error wrapping. #6003 had to fix the ordering in exactly one of them, and the
other two still carry the old behaviour — that is the cost this story removes.

Related, narrower stories: the `is_a?(Hash)` predicate in copy 1 and the
constructor deferral in copy 3 are filed separately; this one is the structural
consolidation.

## Converged shape

One `_assign_attributes` + `assign_nested_parameter_attributes` +
`assign_multiparameter_attributes` trio at the Rails names, in the Rails file,
with the per-key dispatch (`_assign_attribute`, ActiveModel
attribute_assignment.rb:67-75) as its single seam. `#new`, `#assign_attributes`
and `#update` all route through it. `persistence.ts#assignAttributes` currently
inlines `extract`/`execute` rather than calling
`assign_multiparameter_attributes` (:36-39) — that inline goes away too.

## Acceptance criteria

- [ ] One implementation; the other two call sites route through it.
- [ ] Nested deferral, Hash predicate, and error wrapping are identical on all
      three entry paths.
- [ ] `assign_multiparameter_attributes` is called, not inlined.
- [ ] Existing suites (nested-attributes, multiparameter-attributes, dirty,
      forbidden-attributes-protection, base, persistence) stay green.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.

## Triage note (2026-08-18): sequencing against the ActiveModel copy

`activemodel-assign-attribute-still-writes-through-write-attribute` (0023,
~140 LOC) converges the **ActiveModel** `_assignAttribute` arm onto
`attribute_assignment.rb:67-75` (send the setter; stop writing through
`writeAttribute` and sniffing error classes). PR #6216 already did the same for
the ActiveRecord copy.

These were deliberately NOT merged in the 2026-08-18 triage pass — 140 + 350 is
well over the PR LOC ceiling.

Land the ActiveModel one first. It leaves all three copies on the same _shape_,
which turns this story from "reconcile three divergent bodies" into "delete two
of three", and materially lowers its 350 estimate. Re-estimate before claiming.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
