---
title: "Call set_inverse_instance on replace_on_target's inversing arm (collection_association.rb:466)"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CollectionProxy#_wireInverseTarget`
(`packages/activerecord/src/associations/collection-proxy.ts`) is the trails
`target=` inversing caller. PR #6417 converged it onto
`_replaceOnTarget(record, { skipCallbacks: true, replace: true, inversing: true })`,
matching `vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:294`
(`replace_on_target(record, true, replace: true, inversing: true)`).

One deviation survives, guarded by an `if (!inversing)` branch in
`_replaceOnTarget`: Rails calls `set_inverse_instance(record)` unconditionally
at `collection_association.rb:466`, inside `replace_on_target`, on every arm
including the inversing one. trails skips it on the inversing arm because the
reciprocal (record → owner) side is already established by the caller in
`packages/activerecord/src/associations.ts` (`_wireInverseAssociation`, which
calls `proxy._wireInverseTarget(owner)` for a `hasMany` inverse), so re-wiring
would recurse back into `_wireInverseTarget`.

Rails does not recurse here because `set_inverse_instance` on the collection
side resolves the child's _belongs_to_ inverse and calls `inversed_from` on it
(`association.rb:132` → `belongs_to_association.rb`), which sets a target and
replaces keys without calling back into the collection. The trails recursion is
therefore a consequence of where `associations.ts` establishes the reciprocal
side, not of the Rails design.

## Acceptance criteria

1. `_replaceOnTarget` calls `_setCollectionInverseInstance` on every arm, as
   Rails calls `set_inverse_instance(record)` at `collection_association.rb:466`
   — the `if (!inversing)` guard is deleted.
2. The recursion that guard prevents is addressed at its actual source (the
   `associations.ts` / `_wireInverseAssociation` reciprocal write), matching
   Rails' belongs_to `inversed_from` shape rather than re-entering the
   collection.
3. `packages/activerecord/src/associations/inverse-associations.test.ts` and
   `replace-on-target-inversing.trails.test.ts` stay green; no stack overflow on
   the `hasManyInversing` paths.
