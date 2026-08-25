---
title: "read-attribute-miss-path-answers-undefined-not-nomethoderror"
status: ready
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #7015 (`split-read-attribute-between-activemodel-and-activerecord`) converged
`ActiveModel::AttributeMethods#_read_attribute` onto its Rails body:

```ruby
def _read_attribute(attr)
  __send__(attr)
end
```

(`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:556-558`)

trails spells `__send__(attr)` as a property read, because a generated reader is
an accessor property (CLAUDE.md § "Generated attribute readers are properties").
That is right for a name that HAS a reader. It is not yet right for a name that
does not.

In Ruby, `__send__` on an undefined name raises `NoMethodError`, which
`method_missing` intercepts (`attribute_methods.rb:507-522`): if
`matched_attribute_method(name)` matches it dispatches `attribute_missing(match)`,
otherwise it calls `super` and the `NoMethodError` propagates. For the bare
pattern that lands on the private `attribute(attr_name)`
(`activemodel/lib/active_model/attributes.rb:161-163`), gated by
`attribute_method?` — so an undeclared name genuinely raises.

trails has no `method_missing`, so
`(this as Record<string, unknown>)[attr]` simply evaluates to `undefined` for a
name with no reader. The call is silent where Rails is loud.

## Converged shape

`_read_attribute` reaches the same cascade Ruby's `__send__` failure does:
when the name has no reader, route through `matchedAttributeMethod` /
`attributeMissing` and raise where Rails raises, rather than answering
`undefined`.

Note the existing `attributeMissing` surface already exists on `Model`
(`packages/activemodel/src/attribute-methods.ts`), so this is about wiring the
miss path, not inventing one. `packages/activerecord`'s override
(`attribute_methods/read.rb:35-37`) is unaffected — it goes to the attribute
set and has its own `MissingAttributeError` path.

## Acceptance criteria

- `_readAttribute` on a name with no reader raises rather than returning
  `undefined`, matching the `attribute_methods.rb:507-522` cascade.
- Existing callers that legitimately read a declared attribute are unaffected.
- `pnpm vitest run packages/activemodel/src packages/activerecord/src/attribute-methods.test.ts`
- Parity deltas non-negative.
