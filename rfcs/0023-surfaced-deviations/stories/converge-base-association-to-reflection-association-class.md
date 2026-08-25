---
title: "Dispatch Base#association through reflection.association_class instead of the macro switch"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails dispatches the association class off the reflection — there is no macro
switch anywhere (`activerecord/lib/active_record/associations.rb:290-296`):

```ruby
def association(name)
  association = association_instance_get(name)

  if association.nil?
    unless reflection = self.class._reflect_on_association(name)
      raise AssociationNotFoundError.new(self, name)
    end
    association = reflection.association_class.new(self, reflection)
    association_instance_set(name, association)
  end

  association
end
```

`association_class` is a one-line method per reflection subclass
(`reflection.rb`: `HasManyReflection#association_class` returns
`HasManyThroughAssociation` when `options[:through]` else `HasManyAssociation`;
`BelongsToReflection#association_class` returns
`BelongsToPolymorphicAssociation` when `polymorphic?` else
`BelongsToAssociation`; `HasOneReflection` and `ThroughReflection` likewise).

trails already has those methods — `associationClass()` at
`packages/activerecord/src/reflection.ts:1377`, `:1493` and siblings — but
`Base#association` does not use them. It scans `_associations` for the
lightweight definition and hands it to `_buildAssociationInstance`
(`associations/instance-methods.ts:33-50`), which re-derives the same answer
with a `switch (assocDef.macro ?? assocDef.type)` plus `opts.polymorphic` /
`opts.through` checks. PR #6367 made the reflection reach the constructor but
left both the scan and the switch in place, and had to add the `macro ?? type`
fallback precisely because a reflection's `type` is the polymorphic `*_type`
column rather than the macro.

The `_associations` scan is not purely redundant: it carries the
subclass-override ordering ("the most-derived override is the last match"),
which `_reflections` gets for free by being keyed. Converging means confirming
`_reflections` already wins the override the same way, then dropping the scan.

## Converged shape

`association(name)` resolves `_reflectOnAssociation(name)`, raises
`AssociationNotFoundError` when absent, and calls
`reflection.associationClass()`. `_buildAssociationInstance` and its switch go
away, along with the `macro ?? type` fallback.

## Acceptance criteria

- [ ] `Base#association` mirrors `associations.rb:290-296`, dispatching through
      `reflection.associationClass()`.
- [ ] `_buildAssociationInstance`'s macro switch is deleted; the
      `AssociationNotFoundError` raise fires off the missing reflection, not a
      missing `_associations` entry.
- [ ] Subclass-override resolution is covered by a test before the
      `_associations` scan is removed.
- [ ] `pnpm parity:api:calls` / `:args` green; retired rows deleted by hand
      (only-shrink, no `--write`). AR suites pass on all three adapter lanes.
