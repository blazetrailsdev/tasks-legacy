---
title: "saved-change-to-attribute-values-generated-half"
status: done
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6893
claim: "2026-08-22T23:01:54Z"
assignee: "converge-ar-dirty-generic-names-onto-dirty-ts"
blocked-by: null
closed-reason: null
---

# Name and generate the `saved_change_to_attribute` array-returner's per-attribute half

## Context

`activerecord/lib/active_record/attribute_methods/dirty.rb:53-54` declares two
patterns over the same prefix:

```ruby
attribute_method_affix(prefix: "saved_change_to_", suffix: "?", parameters: "**options")
attribute_method_prefix("saved_change_to_", parameters: false)
```

Ruby tells `saved_change_to_name?` (predicate) from `saved_change_to_name`
(the `[old, new]` array) by the `?`. The camel spelling drops it, so both
declarations would produce `savedChangeToName`.

PR #6814 moved the whole dirty cascade onto the pattern machinery and declared
line 53 only — `Base._attributeMethodPatterns` carries
`new AttributeMethodPattern({ prefix: "savedChangeTo", parameters: "**options" })`
(packages/activerecord/src/base.ts, the `static {}` block near the
BeforeTypeCast patterns) — leaving line 54's prefix pattern undeclared, with the
reason in a comment there.

trails spells the two generics `savedChangeToAttribute` (predicate, boolean —
packages/activerecord/src/base.ts:1994, overriding
packages/activemodel/src/model.ts:2251) and `savedChangeToAttributeValues`
(the array — model.ts:2322). The same `?`-collapse produced that `Values`
suffix, so the generated half inherits the question.

The same question applies to `will_save_change_to_attribute?` (dirty.rb:58) vs
`will_save_change_to_attribute` — trails has `willSaveChangeToAttribute` /
`willSaveChangeToAttributeValues` — though AR declares no prefix pattern for
that one.

## Acceptance criteria

- [ ] A spelling for the array-returning `saved_change_to_<attr>` is settled
      against `docs/ruby-ts-conventions.md` (add a rule there if the table does
      not already produce one), and the pair `saved_change_to_attribute?` /
      `saved_change_to_attribute` is spelled consistently at both the generic
      and the generated level.
- [ ] `dirty.rb:54` is declared in `Base._attributeMethodPatterns` and the
      explanatory comment in that `static {}` block is deleted.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative;
      `parity:api:calls` / `:args` add zero rows.
