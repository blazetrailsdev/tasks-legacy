---
title: "insert-record-prefetch-arm-skips-with-cast-value"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`_insert_record` (`vendor/rails/activerecord/lib/active_record/persistence.rb:238-256`)
has a prefetch arm:

```ruby
if prefetch_primary_key? && primary_key
  self.id ||= connection.next_sequence_value(sequence_name)
  attributes_values[primary_key] = _default_attributes[primary_key]
    .with_cast_value(id)
end
```

trails mirrors the branch structurally at
`packages/activerecord/src/persistence.ts:249-260` but assigns the raw sequence
value instead of re-casting it through the default attribute, so
`with_cast_value` is never called — the receipt on `_insertRecord` says exactly
that. It is unobservable today because `prefetchPrimaryKey()` is false for every
adapter trails ships (SQLite, PostgreSQL, MySQL); Rails' arm exists for Oracle
and similar.

## Acceptance criteria

- [ ] The prefetch arm reads `_defaultAttributes[primaryKey]` and re-casts the
      sequence value through it (`withCastValue`) exactly as
      `persistence.rb:245` does, rather than writing the raw value.
- [ ] The `@missingRailsCall with_cast_value` receipt on `_insertRecord`
      (`packages/activerecord/src/persistence.ts`) is deleted, not reworded.
- [ ] A test drives the arm with `prefetchPrimaryKey()` stubbed true and a
      `nextSequenceValue` that returns a value the PK type must cast, asserting
      the cast value reaches the INSERT — failing on the current code.
- [ ] `pnpm parity:api:calls` / `:args` green with no new rows.
