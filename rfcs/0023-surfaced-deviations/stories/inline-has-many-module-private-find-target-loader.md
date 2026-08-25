---
title: "Fold the module-private has_many findTarget loader into HasManyAssociation#findTarget"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 350
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`HasManyAssociation#findTarget` (associations/has-many-association.ts) is a
thin wrapper over a module-private `findTarget(record, assocName, options,
queryExecutor, violatesStrictLoading)` function. That owner/name/options triple
is a trails-only calling convention, kept because the CollectionProxy load path
and the through loaders build an ad-hoc holder for a name/options pair the model
never declared. Rails has one method,
`Association#find_target` (vendor/rails/activerecord/lib/active_record/associations/association.rb:247-273),
reached only through an association instance.

The cost is visible: PR #6472 could not call `violates_strict_loading?` where
Rails calls it (find_target's first statement) because the loader body's cache
and preload lookups — which are really Rails' `find_target?` guard,
association.rb:190 — sit above the raise. The predicate had to be evaluated in
the wrapper and threaded in as a boolean parameter Rails has no counterpart for.
Same for the `queryExecutor` parameter.

## Acceptance criteria

1. `HasManyAssociation#findTarget` holds the body, with no module-private
   `findTarget` beneath it and no owner/name/options parameter triple.
2. The `violatesStrictLoading` and `queryExecutor` parameters are gone;
   `this.isViolatesStrictLoading()` is called in the body, at the raise site.
3. The CollectionProxy and through-loader callers reach it through a real
   association instance rather than `_buildAssociationInstance`'s ad-hoc holder,
   or their remaining need for one is stated with the Rails file:line it stands
   in for.
4. `packages/activerecord/src/associations/` suite stays green.
