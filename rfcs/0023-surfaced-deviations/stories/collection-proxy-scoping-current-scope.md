---
title: "collection-proxy-scoping-current-scope"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while sweeping freeform comments out of
`packages/activerecord/src/associations/**` (story
`strip-freeform-comments-ar-associations`). A deleted comment in
`packages/activerecord/src/associations/association.ts:377` (the `scope()`
branch reading `klass.currentScope()`) recorded that Rails'
`Association#scope` branch

```ruby
elsif (scope = klass.current_scope) && scope.try(:proxy_association) == self
  scope.spawn
```

(`vendor/rails/activerecord/lib/active_record/associations/association.rb:110`)
is unreachable in trails: nothing sets an `AssociationRelation` as
`klass.currentScope`, because `CollectionProxy#scoping` — the Rails caller that
does (`activerecord/lib/active_record/relation.rb`'s `scoping` combined with
`CollectionProxy#scoping`) — is not ported. The TS branch exists but no path
reaches it.

## Acceptance criteria

- [ ] `CollectionProxy#scoping` sets the association relation as the model's
      current scope, matching Rails.
- [ ] `Association#scope`'s `proxy_association == self` branch
      (association.rb:110) is exercised by a test that fails without it.
- [ ] No freeform comment is reintroduced in `association.ts` to describe the
      branch.
