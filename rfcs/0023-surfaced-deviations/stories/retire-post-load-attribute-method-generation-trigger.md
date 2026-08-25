---
title: "defineAttributeMethodsAfterLoad is a trails-only generation trigger; Rails generates on demand from method_missing"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Landed in PR #6788 (RFC 0106,
`attribute-method-generation-driven-from-schema-reflection`). That story moved
generation off schema reflection: `applyColumnsHash` no longer generates, and
`define_attribute_methods` calls `loadSchema` where Rails calls it
(`activerecord/lib/active_record/attribute_methods.rb:114`).

What replaced the eager call is a trails-only hook,
`defineAttributeMethodsAfterLoad` in
`packages/activerecord/src/model-schema.ts`, called at the end of each schema
load (both sync `loadSchema` exits and `loadSchemaFromAdapter`). It carries
`@noRailsEquivalent` because Rails has no such trigger at all: Rails'
`load_schema!` (`model_schema.rb:587-597`) defines nothing, and generation is
demand-driven from the instance —

```ruby
def method_missing(method, ...)          # activemodel/lib/active_model/attribute_methods.rb:474-486
  if respond_to_without_attributes?(method, true)
    super
  else
    match = matched_attribute_method(method.name)
    match ? attribute_missing(match, ...) : super
  end
end
```

trails cannot hook that today: a generated reader is a property, not a method
(CLAUDE.md, "Generated attribute readers are properties"), so there is no miss
to intercept, and the end of a schema load is the nearest demand point.

Two smaller shapes exist only to serve that hook and retire with it:

- `defineAttributeMethods` re-checks `_attributeMethodsGenerated` right after
  `loadSchema` (`packages/activerecord/src/attribute-methods.ts`), because a
  cold load ends in the hook, which runs the whole body first. Rails' second
  check is the post-mutex one at `attribute_methods.rb:108-110`.
- the hook re-clears `_columns`, because generation reads `attribute_names` ->
  `column_names` and re-warms the memo `applyColumnsHash` had just dropped;
  Rails' `load_schema!` leaves `@columns` nil (`model_schema.rb:432-434`).

## Converged shape

Generation becomes demand-driven from the instance, as Rails' is, so
`defineAttributeMethodsAfterLoad` and both of its satellite shapes are deleted
and `define_attribute_methods` is reached only by a real first access (or by an
explicit caller). This is gated on the dispatch layer:
[[attribute-methods-method-missing-dispatch-unported]] has to land first, and
the property-vs-method decision in CLAUDE.md means the trigger will most likely
be the generated-module lookup rather than a literal `method_missing`.

## Acceptance criteria

- [ ] `defineAttributeMethodsAfterLoad` is deleted from `model-schema.ts`, and
      no schema-load path calls `defineAttributeMethods`.
- [ ] The post-`loadSchema` flag re-check in `defineAttributeMethods` is gone;
      only Rails' own two flag checks remain.
- [ ] The `_columns` re-clear is gone.
- [ ] A record that has never been read still answers its attribute accessors.
