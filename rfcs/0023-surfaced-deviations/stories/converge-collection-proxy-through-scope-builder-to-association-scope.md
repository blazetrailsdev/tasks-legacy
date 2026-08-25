---
title: "Delete CollectionProxy#_buildThroughScope in favour of the association's scope"
status: draft
updated: 2026-08-13
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

Surfaced while converging `CollectionProxy#scope` to
`@scope ||= @association.scope` (PR #6492,
`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb:949-951`).

With `scope()` delegating to the association, the proxy's bespoke relation
builders are no longer on the scope path but survive as a parallel seam with no
Rails counterpart:

- `CollectionProxy#_buildThroughScope` (`packages/activerecord/src/associations/collection-proxy.ts`,
  the private method just below `scope()`) — a through-relation builder with a
  JOIN arm (`_routeThroughViaAssociationScope` + `hasManyScope`) and a
  single-column IN-subquery fallback for polymorphic-has_many sources and
  unsaved nested-through.
- The `hasManyScope` import from `apply-association-scope`, still called from
  `_buildThroughScope` and from the proxy's constructor seed path.

Rails has neither: every relation the proxy needs comes from
`Association#scope` → `target_scope.merge!(association_scope)`
(`association.rb:294-307`), with `AssociationScope#scope` (`association_scope.rb:23-38`)
walking the chain and `HasManyThroughAssociation` adding nothing scope-side.

## Converged shape

Callers of `_buildThroughScope` route through `this.scope()` (i.e. the OO
association's `scope()`), and `_buildThroughScope` + the residual
`hasManyScope` call sites in `collection-proxy.ts` are deleted. Any shape that
`AssociationScope` genuinely cannot build today (polymorphic has_many source,
unsaved nested-through) is fixed in `AssociationScope` / `ThroughAssociation`
rather than routed around in the proxy.

## Acceptance criteria

1. `collection-proxy.ts` no longer defines `_buildThroughScope` and no longer
   imports `hasManyScope`.
2. The association suites under `packages/activerecord/src/associations/` pass,
   including `has-many-through-associations.test.ts` and the nested-through /
   disable-joins files.
3. `pnpm parity:api:extra --package activerecord` shows no new extra surface for
   `associations/collection-proxy.ts`.
