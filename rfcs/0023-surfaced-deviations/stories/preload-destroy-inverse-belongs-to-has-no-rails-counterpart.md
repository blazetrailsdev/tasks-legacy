---
title: "preloadDestroyInverseBelongsTo has no Rails counterpart"
status: draft
updated: 2026-07-30
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`removeTargetBang`'s `:destroy` arm calls
`preloadDestroyInverseBelongsTo(this, target)`
(`packages/activerecord/src/associations/has-one-association.ts:666`, helper at
:741) before `target.destroy`. Rails' `remove_target!`
(`vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb:97-102`)
has no such call — it sets `destroyed_by_association` and destroys. The helper
walks the target's belongs_to reflections that key on the same foreign key and
force-loads them so destroy callbacks see a populated back-reference; in Rails
that association is populated by `set_inverse_instance` / normal loading rather
than a bespoke preload pass.

`handleDependency` (:236) makes the same call.

## Acceptance criteria

- [ ] Establish whether the preload compensates for a real inverse-instance gap
      in trails; if so, fix that gap at its source (inverse-of / loading) and
      delete `preloadDestroyInverseBelongsTo`.
- [ ] If it must stay, it is justified at its declaration against the Rails
      behavior it substitutes for.
- [ ] Has-one dependent-destroy callback tests pass unchanged.
