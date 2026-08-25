---
title: "has_one_through: association(name).build skips the through-proxy load that build#{Name} runs"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while shipping `extra-surface-has-one-base-build-hooks-classify`
(PR #5946).

On a has*one_through, `record.association(name).build(...)` and the generated
`record.build#{Name}(...)` accessor take DIFFERENT paths in trails. In Rails
they are the same method — `build*#{name}`is defined as`association(:#{name}).build(\*args, &block)`(builder/singular_association.rb:32-34), so both run`set_new_record`->`replace(record, false)`->`create_through_record`, whose
first act is `through_proxy.load_target`
(has_one_through_association.rb:15-19).

In trails only the accessor performs that join-model load:

- `packages/activerecord/src/associations/has-one-through-association.ts`
  overrides `loadDisplacedForBuild()` to return `null`, so
  `SingularAssociation#build` (singular-association.ts:104-106) issues no load.
- The load is instead driven one level up, by the `build#{name}` accessor
  override in `packages/activerecord/src/associations/builder/has-one.ts`
  (the `findTargetNeeded()` / `loadTargetForBuild()` pre-load).

So `member.association("club").build({...})` on a persisted owner with an
existing but UNLOADED join row skips the reconcile that
`member.buildClub({...})` performs, and can duplicate the join row — the exact
failure the (done) stories `has-one-through-unloaded-displacement-duplicate-join`
and `has-one-through-build-persisted-owner-unloaded-row-reconcile` fixed for
the accessor path only.

The override exists because Rails' `association(:club).build` is
**synchronous** and the ported tests assert that shape —
`vendor/rails/activerecord/test/cases/associations/has_one_through_associations_test.rb:60-70`
(`test_creating_association_builds_through_record`) uses the return value
directly with no await, and our port at
`packages/activerecord/src/associations/has-one-through-associations.test.ts:172-176`
does the same. Making `loadDisplacedForBuild` return the through-proxy load
turned 8 has_one_through tests red when tried in #5946, which is why the split
was left in place there.

This is the real fidelity gap; the split is the symptom. Resolving it means
deciding how a has_one_through `association(name).build` exposes a load JS
cannot run synchronously — the same class of problem RFC 0068 solved for the
has_one writer (`syncWrite` vs the awaitable `writer` / `set#{Name}`).

## Acceptance criteria

- `record.association(name).build(...)` on a has_one_through with a persisted
  owner and an existing UNLOADED join row reconciles that row rather than
  duplicating it, matching `build#{Name}`.
- The chosen surface for the unavoidable async load is documented at the call
  site with its Rails anchor (and an RFC if it is a deliberate deviation, as
  RFC 0068 is for the writer).
- Test names match Rails verbatim; no test renames. If a Rails-ported test
  asserts the synchronous shape, converge the implementation or record why the
  deviation is forced — do not reword the test.
- Ideally the `builder/has-one.ts` `build#{name}` accessor override and the
  through's `loadDisplacedForBuild` null override both disappear, collapsing
  the two build paths back into one as in Rails.

## Notes

Non-trivial: touches the sync/async boundary, so budget for the RFC-0068-style
decision rather than a mechanical fix.
