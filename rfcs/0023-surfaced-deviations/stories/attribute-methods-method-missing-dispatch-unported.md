---
title: "AttributeMethods#method_missing and respond_to?'s attribute arm are unported; one call site open-codes them"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced in PR #6738, converging ActiveModel's `_assignAttribute` onto
`attribute_assignment.rb:67-75`.

Rails' `AttributeMethods` overrides two dispatch entry points on the instance:

```ruby
def method_missing(method, ...)           # attribute_methods.rb:507-514
  if respond_to_without_attributes?(method, true)
    super
  else
    match = matched_attribute_method(method.name)
    match ? attribute_missing(match, ...) : super
  end
end

def respond_to?(method, include_private_methods = false)   # :528-538
  if super
    true
  elsif !include_private_methods && super(method, true)
    false
  else
    !matched_attribute_method(method.to_s).nil?
  end
end
```

Those two are what make an attribute method work when it has NOT been generated
— most visibly after `undefine_attribute_methods`
(`attribute_methods.rb:466-478`), which strips the generated module and leaves
`method_missing` to carry every read and write.

trails ports the pieces — `matchedAttributeMethod`
(`packages/activemodel/src/attributes.ts:347`), `attributeMissing`
(`packages/activemodel/src/attribute-methods.ts:169`),
`isRespondToWithoutAttributes` (`:194`) — but not the two dispatchers, because
TS has no `method_missing`. PR #6738 needed the `method_missing` behaviour for
`_assignAttribute`'s rescue arm and open-coded it AT THAT ONE CALL SITE
(`packages/activemodel/src/attribute-assignment.ts`, the
`matchedAttributeMethod` → `attributeMissing` rungs), which is why assignment
survives `undefine_attribute_methods`. Every OTHER path that would reach an
ungenerated attribute method in Rails still gets nothing: reads, predicates, and
the dirty-tracking suffix methods all resolve to `undefined` rather than
dispatching through `attribute_missing`.

`respond_to?`'s attribute arm (`:536`) is also unported — `Model#respondTo`
does not consult `matched_attribute_method`, so it answers false for a name the
receiver would in fact serve.

## Converged shape

One dispatcher the ungenerated-name paths share, rather than a ladder per call
site. TS cannot hook property access on a plain class, so the honest options are
a `Proxy`-based host (heavy, and it changes identity semantics) or an explicit
`attributeMethodMissing(name, ...args)` on `Model` that ports :507-514 verbatim
and that each call site needing ungenerated-name dispatch routes through —
`_assignAttribute` first, replacing its inline rungs.

`respondTo` gains the `:536` arm regardless; that one needs no language
workaround, it is a missing branch.

## Acceptance criteria

- [ ] `method_missing`'s body (:507-514) exists once, at the Rails name, and
      `_assignAttribute`'s inline `matchedAttributeMethod`/`attributeMissing`
      rungs route through it rather than duplicating it.
- [ ] `Model#respondTo` consults `matched_attribute_method` (:536); a test pins
      an ungenerated attribute name answering true.
- [ ] A test reads an attribute after `undefine_attribute_methods` and gets the
      value, matching Rails; verified failing on the baseline.
- [ ] No test renames.
