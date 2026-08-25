---
title: "Split composite-PK id/id= arms out of PrimaryKey into CompositePrimaryKey"
status: draft
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
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

Rails' `ActiveRecord::AttributeMethods::PrimaryKey#id=` is one line
(`vendor/rails/activerecord/lib/active_record/attribute_methods/primary_key.rb:29`):

```ruby
def id=(value)
  _write_attribute(@primary_key, value)
end
```

The composite-primary-key behaviour (the `TypeError unless value is Enumerable`
raise and the `@primary_key.zip(value)` nil-padding spread) lives in
`ActiveRecord::CompositePrimaryKey#id=`, a separate module Rails mixes in only
for composite-keyed models.

trails collapses all of it into one branchy writer:
`packages/activerecord/src/attribute-methods/primary-key.ts` `writeId`, now
reached through the `PrimaryKey` class module's `set id()` (landed in #5413,
which moved the body verbatim and did not change behaviour). The same function
carries a third arm Rails does not have at all — the `pk == null` key-less-model
branch that routes to the public `writeAttribute("id", value)`.

The reader `readId` has the mirror-image problem: Rails' `#id` is
`_read_attribute(@primary_key)` with the array-mapping in `CompositePrimaryKey#id`.

## Acceptance criteria

- `PrimaryKey#id` / `#id=` reduce to their one-line Rails bodies.
- The composite arms move to a `CompositePrimaryKey` module mixed in for
  composite-keyed models, matching Rails' file layout.
- The key-less-model arm is either justified at the call site as a deviation
  with the Rails behaviour it stands in for, or removed if `_write_attribute`
  already covers it.
- `primary-keys.test.ts`, `primary-keys.trails.test.ts` stay green;
  parity:api and parity:test deltas non-negative.
