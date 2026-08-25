---
title: "Retire CollectionProxy#_pushThrough for proxy_association.concat"
status: draft
updated: 2026-08-13
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

`packages/activerecord/src/associations/collection-proxy.ts#_pushThrough` has no
Rails counterpart — its own JSDoc opens "Rails has no `_pushThrough`". Rails'
`CollectionProxy#<<` is `proxy_association.concat(records)`
(`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb:1053`),
and every join-row decision lives on `HasManyThroughAssociation#concat_records` /
`#insert_record` (has_many_through_association.rb:24-49).

After #6461 (through-`create` delegated to the association) and PR #6464 (the
dead `skipCallbacks` arm and its `_throughTransaction` helper deleted),
`_pushThrough` has shrunk to exactly:

    const assoc = this._throughAssociation();
    const previousThroughScope = assoc._throughScope;
    if (throughScope != null) assoc._throughScope = throughScope;
    try { await assoc.concat(...records); } finally {
      assoc._throughScope = previousThroughScope;
    }
    this._invalidateAssociationIds();

Two callers remain, both passing only `records` — so the `throughScope`
parameter is dead as well. What is left is `concat` plus a trails-only
`_invalidateAssociationIds()` memo sweep.

## Converged shape

Retire `_pushThrough`: call `this._collectionAssociation().concat(...)` at the
two call sites, as collection_proxy.rb:1053 does, and give the memo sweep a home
that matches Rails — `insert_record` ends in `reset_scope`
(collection_association.rb), which is what `_invalidateAssociationIds` is
standing in for. Drop the dead `throughScope` parameter with it.

## Acceptance criteria

- [ ] `_pushThrough` and its dead `throughScope` parameter are gone; both call
      sites read as Rails' `proxy_association.concat(records)`.
- [ ] The `_associationIds` / `_offsetMemo` invalidation still happens on the
      Rails-shaped path (`resetScope`), not from a proxy-local wrapper.
- [ ] `pnpm parity:api:extra --package activerecord` does not regress;
      `associations/` suites and `through-push-routes-to-insert-record.trails.test.ts`
      green.
