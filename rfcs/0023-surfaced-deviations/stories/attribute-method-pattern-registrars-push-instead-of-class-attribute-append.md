---
title: "attribute_method_suffix/prefix/affix push instead of class_attribute +="
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Rails' `attribute_method_prefix` / `attribute_method_suffix` / `attribute_method_affix`
(`activemodel/lib/active_model/attribute_methods.rb:106-107`, `:140-141`) append
with `+=` on a `class_attribute`:

```ruby
self.attribute_method_patterns += suffixes.map! { |suffix| ... }
```

`class_attribute` `+=` is copy-on-write — the receiving class gets its OWN array
and no ancestor is mutated. trails' ports
(`packages/activemodel/src/attribute-methods.ts`) instead do
`this._attributeMethodPatterns.push(...)` behind an `ensureOwnPatterns(this)`
helper, which is a hand-rolled stand-in for the same semantics and does not read
as the Ruby.

The distinction is load-bearing: registering a suffix on a subclass must not add
it to `Model._attributeMethodPatterns`, or every ActiveModel class in the process
starts generating that suffix. PR 6779 sidestepped the question by assigning a
fresh array on `Base` directly (`packages/activerecord/src/base.ts`,
`static override _attributeMethodPatterns = [...Model._attributeMethodPatterns, ...]`)
rather than calling `attributeMethodSuffix`, precisely because calling it on
`Base` would have gone through the push path.

## Converged shape

The three registrars assign rather than push:
`this._attributeMethodPatterns = [...this._attributeMethodPatterns, ...new]`,
which IS `+=` and makes `ensureOwnPatterns` unnecessary — delete it.

`Base`'s `BeforeTypeCast`/`ForDatabase` registration then becomes the Rails
spelling, an `attributeMethodSuffix("BeforeTypeCast", "ForDatabase",
{ parameters: false })` call mirroring the `included do` block at
`activerecord/lib/active_record/attribute_methods/before_type_cast.rb:32`,
instead of a hand-built static array.

## Acceptance criteria

- [ ] `attributeMethodPrefix` / `attributeMethodSuffix` / `attributeMethodAffix`
      assign a new array, mirroring `+=`.
- [ ] `ensureOwnPatterns` is deleted.
- [ ] `Base` registers its two suffixes through `attributeMethodSuffix`, not a
      literal `_attributeMethodPatterns` array.
- [ ] Registering a suffix on a subclass leaves `Model._attributeMethodPatterns`
      untouched (covered by a test).
