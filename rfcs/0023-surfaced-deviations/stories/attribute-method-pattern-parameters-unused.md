---
title: "AttributeMethodPattern#parameters is stored but never used when defining attribute methods"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `AttributeMethodPattern#parameters` is stored but never used when defining attribute methods

## Context

PR #6557 threaded Rails' `parameters:` keyword through
`attribute_method_prefix` / `attribute_method_suffix` into
`AttributeMethodPattern` (attribute_methods.rb:106,140,476), converging the
constructor-argument rows. The field is now populated correctly — and read by
nothing.

In Rails, `parameters` is what shapes the GENERATED method's signature.
`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:320-`
(`define_attribute_method_pattern`) passes `pattern.parameters` into the
`class_eval`'d definition, so a pattern declared with
`attribute_method_suffix "_changed?", parameters: false` generates a
zero-arity method rather than a `...`-forwarding one.

trails' `defineAttributeMethodPattern`
(`packages/activemodel/src/attribute-methods.ts`) defines every generated method
with a fixed signature and never consults `pattern.parameters`, so the keyword
is inert.

## Converged shape

`defineAttributeMethodPattern` honours `pattern.parameters` when defining the
method — `"..."` (the default) forwards arguments, `false` defines a
no-argument method — matching attribute_methods.rb:320-.

## Acceptance criteria

- [ ] The generated method's argument forwarding follows `pattern.parameters`.
- [ ] A ported Rails test exercising `parameters: false` passes and fails on the
      current baseline.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
