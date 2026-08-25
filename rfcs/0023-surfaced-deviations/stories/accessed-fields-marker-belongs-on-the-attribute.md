---
title: "accessed_fields marks reads at call sites instead of on the Attribute"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails marks an attribute as read on the `Attribute` object itself: `Attribute#value`
memoizes `@value` (`vendor/rails/activemodel/lib/active_model/attribute.rb:41-47`),
and `has_been_read?` is `defined?(@value)`. `AttributeSet#accessed`
(`attribute_set.rb:38`) then selects the attributes that were read. Because the
marker lives on the value object, **every** read path feeds `accessed_fields` —
the generated accessor, `read_attribute`, `_read_attribute`, and a direct
`fetch_value`.

trails instead keeps a `_accessedFields` Set on the record and adds to it at
chosen call sites: `Model#readAttribute`
(`packages/activemodel/src/model.ts`, pre-existing) and, since PR #6717, AR's
generated reader (`packages/activerecord/src/attribute-methods/read.ts`).
`_readAttribute` and `fetchValue` stay silent, so an internal read undercounts
`accessed_fields` relative to Rails.

Surfaced in PR #6717: widening the marker to `_readAttribute` was tried and
reverted — it made every internal read feed the Set, which reordered it and broke
record equality in `relations.test.ts` ("loading with one association"). That is
evidence the Set-on-the-record shape is the problem, not the call-site choice.

## Converged shape

Move the marker to where Rails keeps it: `Attribute` gains the read flag
(`hasBeenRead`), `AttributeSet` gains `accessed()` returning the read attributes'
names, and `accessedFields` (`packages/activerecord/src/attribute-methods.ts`)
answers from the attribute set rather than from a Set maintained by callers.
The per-call-site `_accessedFields.add(...)` lines then disappear.

## Acceptance criteria

- [ ] `Attribute` carries the read marker, set where the value is materialised
      (attribute.rb:41-47).
- [ ] `AttributeSet#accessed` mirrors attribute_set.rb:38; `accessedFields` reads
      through it and `_accessedFields` is gone.
- [ ] `accessed_fields` counts reads through `_read_attribute` and a generated
      reader, not only through `read_attribute` — with a test that reads an
      attribute by each path.
- [ ] `relations.test.ts` "loading with one association" still passes (record
      equality must not depend on read order).
