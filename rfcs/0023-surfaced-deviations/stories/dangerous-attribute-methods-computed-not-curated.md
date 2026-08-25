---
title: "Compute dangerous_attribute_methods from Base instead of a hand-curated literal set"
status: draft
updated: 2026-08-14
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

`ActiveRecord::AttributeMethods.dangerous_attribute_methods`
(`vendor/rails/activerecord/lib/active_record/attribute_methods.rb:30-37`) is
computed:

```ruby
def self.dangerous_attribute_methods # :nodoc:
  @dangerous_attribute_methods ||= (
    Base.instance_methods +
    Base.private_instance_methods -
    Base.superclass.instance_methods -
    Base.superclass.private_instance_methods
  ).map(&:to_s).to_set.freeze
end
```

trails (`packages/activerecord/src/attribute-methods.ts`,
`dangerousAttributeMethods`) hand-maintains a literal `Set` of ~40 names with a
comment admitting it approximates the computation. PR #6543 made the set
load-bearing — `isInstanceMethodAlreadyImplemented` now raises
`DangerousAttributeError` from it — and had to hand-add `createOrUpdate` and
`isFrozen` to make the Rails test
(`activerecord/test/cases/attribute_methods_test.rb:717-724`) pass, which is the
maintenance burden the curated list guarantees: every AR instance method added
since is silently missing, so an attribute named after it generates an accessor
that shadows Active Record.

## Converged shape

Compute the set from `Base`'s own prototype: the own + inherited property names
of `Base.prototype` minus those of `Object.prototype` (JS' stand-in for
`Base.superclass`), memoized once as Rails memoizes it. Drop the literal list.
Names Rails gets from Ruby but JS has no analogue for (`hash`, `frozen?` where
trails spells it `isFrozen`) are covered by the computation or by an explicit,
cited exception — not by an unexplained literal.

## Acceptance criteria

- [ ] `dangerousAttributeMethods()` is computed from `Base.prototype`, not a
      literal list, and stays memoized.
- [ ] The Rails test at attribute_methods_test.rb:717-724 still passes, and
      `save` / `createOrUpdate` / `dup` / `isFrozen` are in the computed set
      without being named in the source.
- [ ] No canonical model regresses: the whole AR suite is green on all three
      adapters (the computed set is strictly larger, so a column named after a
      newly-covered method will now raise, which is Rails' behaviour).
