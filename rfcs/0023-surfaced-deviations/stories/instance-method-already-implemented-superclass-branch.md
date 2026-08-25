---
title: "Port instance_method_already_implemented?'s superclass branch instead of a bare prototype probe"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::AttributeMethods::ClassMethods#instance_method_already_implemented?`
(`vendor/rails/activerecord/lib/active_record/attribute_methods.rb:165-179`) has
two halves. PR #6543 ported the first — the `dangerous_attribute_method?` raise
at :166-168 — and wired the predicate onto `Base` so
`define_attribute_method_pattern` reaches it. The second half is still
approximated:

```ruby
if superclass == Base
  super
else
  # If ThisClass < ... < SomeSuperClass < ... < Base and SomeSuperClass
  # defines its own attribute method, then we don't want to override that.
  defined = method_defined_within?(method_name, superclass, Base) &&
    ! superclass.instance_method(method_name).owner.is_a?(GeneratedAttributeMethods)
  defined || super
end
```

trails (`packages/activerecord/src/attribute-methods.ts`,
`isInstanceMethodAlreadyImplemented`) returns `methodName in this.prototype`
instead. That is close because the generated module is spliced into the
prototype chain, but it cannot distinguish a real method an intermediate
superclass defines from a _generated accessor_ it inherited — Rails explicitly
excludes the latter via the `GeneratedAttributeMethods` owner check, so a
grandchild class regenerates its own accessor where trails keeps the ancestor's.
It also never calls the ActiveModel base implementation (`super`), which
consults `generated_attribute_methods.method_defined?`.

`isMethodDefinedWithin` already exists at `attribute-methods.ts:542` (the port of
`method_defined_within?` at `attribute_methods.rb:187-197`) and is unused by this
path.

Surfaced by review of PR #6543.

## Converged shape

`isInstanceMethodAlreadyImplemented` mirrors :165-179 branch for branch: the
raise, then the `superclass == Base` split, with `isMethodDefinedWithin` and an
owner check against `GeneratedAttributeMethods` in the else arm, and the
ActiveModel implementation standing in for `super` in both arms.

## Acceptance criteria

- [ ] `isInstanceMethodAlreadyImplemented` has Rails' two branches, in Rails'
      order, using `isMethodDefinedWithin`.
- [ ] A grandchild class whose intermediate superclass has only a _generated_
      accessor for the name regenerates its own, while one whose superclass
      defines a real method does not.
- [ ] `pnpm parity:api:calls` shows the `instance_method_already_implemented?`
      row set shrink (or stay flat), never grow.
