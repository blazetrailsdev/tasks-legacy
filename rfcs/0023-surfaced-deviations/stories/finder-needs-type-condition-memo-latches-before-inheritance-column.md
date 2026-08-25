---
title: "finder_needs_type_condition? memoizes true before consulting the inheritance column"
status: draft
updated: 2026-08-22
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

# `finder_needs_type_condition?` memoizes `true` before consulting the inheritance column

## Context

Rails' `finder_needs_type_condition?`
(`activerecord/lib/active_record/inheritance.rb:92-95`) memoizes the answer, and
the thing it memoizes — `descends_from_active_record?`
(`inheritance.rb:78-88`) — reads `inheritance_column` on every arm that can
answer false:

```ruby
def descends_from_active_record?
  if self == Base
    false
  elsif superclass.abstract_class?
    superclass.descends_from_active_record?
  else
    superclass == Base || !columns_hash.include?(inheritance_column)
  end
end
```

trails' `isFinderNeedsTypeCondition`
(`packages/activerecord/src/inheritance.ts:706-722`) latches `true` on the
`!isDescendsFromActiveRecord()` branch **before** the column is consulted, and
the `inheritanceColumn === null` arm below it can only ever memoize `false`. So
once the memo is `true`, clearing the inheritance column never revises it.

Measured on the canonical `Firm` / `companies` fixtures (PR #6852):

```text
{ before: true, col: null, after: true, result: 'THREW: Cannot build type condition without an inheritance column' }
```

Two trails-only facts meet here:

1. the latch above, and
2. `typeCondition` (`inheritance.ts:936-940`) **raises** `ActiveRecordError`
   when `inheritanceColumn === null`, where Rails' `type_condition`
   (`inheritance.rb:322`) just builds `table[inheritance_column]` — with a nil
   column that is a junk predicate, not an error.

PR #6852 had to add `if (this.inheritanceColumn === null) return relation;` inside
`Base.relation()`'s type-condition arm (base.ts:2186-2192) to keep a Rails
no-op from becoming a throw. That guard is a symptom, not the fix — it sits
inside a body that otherwise mirrors core.rb:431-435 exactly.

Note Rails' public setter coerces: `real_inheritance_column=`
(`activerecord/lib/active_record/model_schema.rb:325-327`) does `.to_s`, so
`self.inheritance_column = nil` sets `""` in Rails and `nil` is unreachable
through the documented setter. trails' `null` is its own established
"STI disabled" sentinel (`inheritance.ts:718`, `model-schema.ts:1579`), which is
why the state is reachable here and not there.

## Converged shape

Decide the sentinel question first, then remove the symptom:

- either `inheritanceColumn = null` revises the memo (Rails' `descends_from_active_record?`
  re-reads the column, so a class with STI disabled is `descends` → finder-needs
  false), which makes `relation()`'s guard dead and lets it be deleted;
- or `typeCondition` stops raising and mirrors `table[inheritance_column]`,
  which does the same.

Either way `Base.relation()` goes back to core.rb:431-435 verbatim, with the
`inheritanceColumn === null` line deleted and the regression test in
`base.trails.test.ts` ("skips the type condition when the inheritance column is
gone") still green.

## Acceptance criteria

- [ ] `Base.relation()` no longer carries the `inheritanceColumn === null`
      early return; its body is core.rb:431-435 verbatim.
- [ ] `Base.relation with a cleared inheritance column (trails)` in
      `packages/activerecord/src/base.trails.test.ts` still passes.
- [ ] The chosen fix is cited against `inheritance.rb:78-95` or `:322`, whichever
      it converges.
- [ ] `parity:api:calls` green; STI suites pass on all three adapters.
