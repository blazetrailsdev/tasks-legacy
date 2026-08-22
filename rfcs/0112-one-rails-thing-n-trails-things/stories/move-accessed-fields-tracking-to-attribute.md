---
title: "Track accessed fields on the attribute (has_been_read?), not a Set on Model"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 90
priority: 40
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`accessedFields` bookkeeping is in the wrong layer. Rails records reads on the
attribute itself: `ActiveModel::Attribute#value` memoizes and flips
`has_been_read?` (activemodel/lib/active_model/attribute.rb), `AttributeSet#accessed`
selects those names (activemodel/lib/active_model/attribute_set.rb), and
`ActiveRecord::AttributeMethods#accessed_fields` is just `@attributes.accessed`
(activerecord/lib/active_record/attribute_methods.rb:453-455).

trails instead keeps a `_accessedFields: Set<string>` on the model
(`packages/activemodel/src/model.ts:1540`) and adds to it inside
`Model#readAttribute` (~:1677). Reads that do not go through that one method —
the generated per-attribute getters, `_readAttribute`, any AR-level
`readAttribute` — record nothing.

Surfaced in PR #6068: wiring AR's ported `readAttribute` (read.rb:31-34) onto
`Base` reds the `accessed_fields` test purely because the bookkeeping rides on
the ActiveModel method rather than on the attribute. See
[[wire-ar-read-write-attribute-onto-base]].

## Converged shape

- `Attribute` gains `hasBeenRead()` set by `value` (attribute.rb).
- `AttributeSet.accessed()` returns the names whose attribute has been read
  (attribute_set.rb).
- `accessedFields` (packages/activerecord/src/attribute-methods.ts:156) returns
  `this._attributes.accessed()`; `Model#readAttribute` stops mutating a Set and
  `_accessedFields` is deleted.

## Acceptance criteria

- [ ] `_accessedFields` no longer exists; reads through generated getters and
      `_readAttribute` are counted, matching Rails.
- [ ] `accessed_fields` tests stay green.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
