---
title: "Delete the new-owner `1=0` seed rebase apparatus; Rails' reset_scope call sites cover it"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`_seededNoneNewOwner`, `_seedWherePredicates` (`relation.ts:369,375`),
`rebaseNewOwnerSeed` (`associations/new-owner-seed-rebase.ts`),
`AssociationRelation#_maybeRebaseAssociationSeed` and
`CollectionProxy#_maybeRebaseProxySeed` / `#_isEmptyRelation` have no Rails
counterpart. They exist to rescue a relation that was built off a new owner's
`1=0` NullRelation seed and then read after the owner was saved.

Rails needs none of it. Its relation is always spawned from a freshly built
scope, and the two memos that could go stale are dropped explicitly:
`reader` ends in `@proxy.reset_scope`
(`vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:41`,
which clears `@scope`, `@offsets`, `@take` — `collection_proxy.rb:1112-1116`),
and `save_collection_association` calls `association.reset_scope` "to
reconstruct the scope now that we know the owner's id"
(`vendor/rails/activerecord/lib/active_record/autosave_association.rb:428` →
`association.rb:119-121`). trails ports both call sites
(`associations.ts:1802`, `autosave-association.ts:243`).

PR #6595 (`collection-proxy-delegate-query-methods-to-scope`) moved the marker
onto the scope returned by `CollectionProxy#scope`, since relations are now
spawned there rather than off the proxy, but did not remove the apparatus.

## Converged shape

Delete the rebase apparatus and let Rails' two `reset_scope` call sites carry
it. A relation captured before the owner's save is stale in Rails too, so the
tests that pin the rebase
(`collection-proxy.test.ts` "CollectionProxy — mutated finder requery on stale
new-owner seed" and "… mutation terminals invoked on the PROXY itself …") are
pinning trails-only behavior and go with it; check each against the Rails
counterpart before deleting.

Depends on `collection-proxy-delegate-query-method-bangs-to-scope` (the
`_cpMutated` half of the same machinery).

## Acceptance criteria

- [ ] `_seededNoneNewOwner`, `_seedWherePredicates`, `rebaseNewOwnerSeed`,
      `_maybeRebaseAssociationSeed`, `_maybeRebaseProxySeed` and the
      `CollectionProxy#_isEmptyRelation` / `AssociationRelation#_isEmptyRelation`
      rebase overrides are deleted.
- [ ] `CollectionProxy#scope` is `collection_proxy.rb:1119-1121` verbatim, with
      no marking.
- [ ] Association tests stay green on SQLite, PostgreSQL and MySQL/MariaDB.
