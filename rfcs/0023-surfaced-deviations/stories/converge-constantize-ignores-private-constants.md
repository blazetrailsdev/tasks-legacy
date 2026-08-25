---
title: "constantize must ignore private constants, as Object.const_get does"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Raised in review of PR #5503.

`constantize` (`packages/activesupport/src/inflector.ts:261-265`) raises
`private constant <name> referenced` for a name marked by `privateConstant`.
Ruby's `Object.const_get` — which `Inflector.constantize` _is_
(`activesupport/lib/active_support/inflector/methods.rb:289-291`, body
`Object.const_get(camel_cased_word)`) — does not do this. `private_constant`
blocks only the literal `A::B` scope-operator reference.

Verified on ruby 3.3.11:

```ruby
class C; class D; end; private_constant :D; end
Object.const_get("C::D")   # => C::D
C.const_get(:D)            # => C::D
C::D                       # NameError: private constant C::D referenced
```

PR #5514 introduced the enforcement on the premise that
`Object.const_get("Country::HABTM_Treaties")` raises and that Rails' habtm
association survives by "resolving through the reflection, not the constant
table". Both halves are wrong: `const_get` succeeds, and Rails' middle
reflection sets only `class_name`
(`associations/builder/has_and_belongs_to_many.rb:73`), resolving the join model
back out of the constant table through `compute_type`.

The divergence became load-bearing once PR #5503 routed Active Record's model
resolution through `constantize`: the habtm middle reflection's `klass` started
raising, `throughLabelAssociations` (`fixtures.ts:413`) stopped recognising the
habtm fixture label as an association, and fixture loading failed on all four
adapters with `table developers has no column named sharedComputers`. #5503
worked around it by carrying the join model on the middle reflection as
`anonymousClass` — an option Rails does not set — rather than changing
activesupport mid-review.

## Acceptance criteria

- `constantize` / `safeConstantize` ignore `_privateConstants`, matching
  `Object.const_get`. If a privacy concept is kept at all, it belongs to
  whatever models _literal_ reference resolution, which trails has no analogue
  for — decide explicitly and record the decision.
- `private-constant.trails.test.ts` is updated to Ruby's actual semantics (its
  "constantize raises on a private constant" and "safe constantize returns
  undefined for a private constant" cases encode the wrong behaviour), or the
  file retires with `privateConstant` if the mark has no remaining consumer.
- The `anonymousClass: JoinModel` workaround is REMOVED from
  `associations/builder/has-and-belongs-to-many.ts` middleOptions, restoring
  Rails' exact `middle_options` shape, and habtm fixture loading stays green on
  all four adapters (`test-fixtures.test.ts`, the `sharedComputers` label).
- PR #5503's follow-on commit that made the private-constant error a `NameError`
  (so `safeConstantize` swallowed it) is revisited — it exists only to keep the
  enforcement's blast radius contained.
