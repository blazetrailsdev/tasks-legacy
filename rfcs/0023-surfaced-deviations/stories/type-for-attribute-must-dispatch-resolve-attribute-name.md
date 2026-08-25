---
title: "typeForAttribute dispatches through the class resolveAttributeName override"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveModel::AttributeRegistration::ClassMethods#type_for_attribute`
(vendor/rails/activemodel/lib/active_model/attribute_registration.rb:43-50)
calls the private `resolve_attribute_name`, which Ruby dispatches to
`ActiveModel::AttributeMethods`' alias-aware override
(vendor/rails/activemodel/lib/active_model/attribute_methods.rb:396-398)
because AttributeMethods is included after AttributeRegistration.

trails' `typeForAttribute` in
`packages/activemodel/src/attribute-registration.ts` instead calls that
file's own identity implementation directly
(`resolveAttributeName.call(this, name)`), so an ActiveModel-only class
with `alias_attribute` gets the un-resolved name. PR #6480 converged the
same dispatch bug in `attribute` and `decorateAttributes` (both now call
`this.resolveAttributeName(name)`) but deliberately left `typeForAttribute`
alone to keep the diff scoped; AR's `Base.typeForAttribute` resolves aliases
by hand, which masks it there.

## Converged shape

`typeForAttribute` calls `this.resolveAttributeName(name)` like its two
siblings, and AR's hand-rolled alias lookup in `base.ts` `typeForAttribute`
is re-checked for redundancy once it does.

## Acceptance criteria

- [ ] `typeForAttribute` dispatches through the class-level
      `resolveAttributeName`, not the file-local identity function.
- [ ] An ActiveModel-only class aliasing an attribute resolves its type
      through the alias.
- [ ] No baseline row added or widened.
