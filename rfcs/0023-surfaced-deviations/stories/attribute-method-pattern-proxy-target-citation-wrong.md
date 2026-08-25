---
title: "Correct the attribute_methods.rb:552 proxy-target citation"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 10
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Correct the `attribute_methods.rb:552` proxy-target citation

## Context

`packages/activemodel/src/attribute-methods.ts:79` documents
`AttributeMethodPattern`'s affix join as:

> Ruby joins `"#{prefix}#{attr_name}#{suffix}"` over snake_case affixes
> (attribute_methods.rb:552)

`vendor/rails/activemodel/lib/active_model/attribute_methods.rb:552` is
`def missing_attribute(attr_name, stack)`. The line the comment means is **481**:

```ruby
@proxy_target = "#{@prefix}attribute#{@suffix}"   # :481
```

with the reader at `:472` (`attr_reader :prefix, :suffix, :proxy_target,
:parameters`). The generated-name join the sentence describes is
`method_name`, built from the same two affixes.

Found while auditing #6821's own citations, which had copied `:552` into
`packages/activerecord/src/base.ts`. That copy is fixed; the original is not.

This matters more than a typo: the proxy-target derivation is the cited
justification for the forced `savedChangeToAttribute` spelling
([[converge-ar-dirty-generic-names-onto-dirty-ts]]), so the line a reader lands
on decides whether that justification checks out.

## Converged shape

Point the comment at `:481` (and `:472` for the reader), and say `proxy_target`
rather than only describing the generated-name join, since both are built from
the same affix pair and only one of them is what the following sentence explains.

## Acceptance criteria

- [ ] `attribute-methods.ts:79` cites the line that actually carries the join.
- [ ] No other `attribute_methods.rb:552` citation survives in the tree.
