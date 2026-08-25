---
title: "Drop define_attribute_method_pattern's extra prototype guard arm now that AR's predicate dispatches"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
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

`define_attribute_method_pattern`
(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:319-331`) guards
on exactly one predicate:

```ruby
if instance_method_already_implemented?(public_method_name)
  return unless override
end
```

trails (`packages/activemodel/src/attribute-methods.ts`,
`defineAttributeMethodPattern`) guards on that predicate **or** a second arm,
`publicMethodName in this.prototype`. The extra arm was documented at the call
site as standing in for ActiveRecord's override of the predicate, which trails
could not reach because the call site imported the ActiveModel function
statically.

PR #6543 removed that reason: the predicate is now dispatched through the class
and ActiveRecord's override is registered on `Base`
(`activerecord/lib/active_record/attribute_methods.rb:165`, `base.ts`). The
prototype arm is therefore redundant for ActiveRecord hosts, and for plain
ActiveModel hosts it is a deviation — Rails' ActiveModel predicate
(`attribute_methods.rb:404-406`) consults only `generated_attribute_methods`, so
a method defined in the model's own class body should NOT stop generation there.

Surfaced by review of PR #6543.

## Converged shape

The guard is the single dispatched predicate call, exactly as :326. Whatever the
prototype arm was really protecting moves into the host's own predicate — where
Rails puts it — or is shown to be unnecessary by the suite.

## Acceptance criteria

- [ ] `defineAttributeMethodPattern`'s guard is one predicate call, no `||`.
- [ ] `alias attribute respects user defined method` /
      `... in parent classes` (activemodel attribute-methods.test.ts) still pass.
- [ ] AR attribute-method suites green on all three adapters.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
