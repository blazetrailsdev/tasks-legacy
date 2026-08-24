---
title: "build-record-await-deferred-initialize"
status: closed
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
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
closed-reason: "Converged in PR #7005: buildRecord parks the deferred initializeAttributes on the record via parkNestedReaderLoad so save drains it, keeping Rails' yield-then-return order (association.rb:383-388). No awaited buildRecord needed."
---

## Context

Rails' `Association#build_record` runs `initialize_attributes(record, attributes)`
and then `yield(record)` inside the block it passes to
`reflection.build_association`, so both have completed by the time `build_record`
returns:

```ruby
# activerecord/lib/active_record/associations/association.rb:383-388
def build_record(attributes)
  reflection.build_association(attributes) do |record|
    initialize_attributes(record, attributes)
    yield(record) if block_given?
  end
end
```

Callers rely on that: `SingularAssociation#build` calls `set_new_record(record)`
immediately after (`singular_association.rb:29-32`), and `_create_record` calls
`record.save` (`singular_association.rb:67-70`,
`collection_association.rb:117-122`, `:362-365`).

In trails `_assignAttributes` is `Promise<void> | void`
(`packages/activemodel/src/model.ts:158`) because an association or nested writer
it reaches can owe I/O. `Association#initializeAttributes`
(`packages/activerecord/src/associations/association.ts:698`) therefore returns
`Promise<void> | void` too, and `buildRecord`
(`packages/activerecord/src/associations/association.ts:817`) yields the caller
block from that continuation when the assign defers. A JS constructor cannot
await, and `buildRecord` returns a synchronous `Base | null`, so in the deferred
case `buildRecord` returns before the block has run — `set_new_record`/`save`
follow while the assign is still in flight.

The attributes come from `scope_for_create`
(`vendor/rails/activerecord/lib/active_record/relation.rb:1231-1235`): the
`where_clause.to_h(equality_only: true)` half is column names only, but the
`create_with_value` half is a user hash that can name an association writer, so
the deferred path cannot be ruled out at the call site.

Surfaced in review on PR #7005 (the RFC 0061 fix for the `no-floating-promises`
red at `association.ts:710`), which converged the
`_assignAttributes`-before-`setInverseInstance` ordering but could not close this
half without a cross-cutting async change.

## Acceptance criteria

- `buildRecord` preserves Rails' initialize → yield → return ordering when
  `_assignAttributes` defers: the caller block has run before `buildRecord`'s
  value reaches `set_new_record` / `save`.
- The seven `buildRecord` call sites are converged with it:
  `association.ts:801`, `collection-association.ts:375`, `:474`,
  `singular-association.ts:82`, `:390`,
  `has-many-through-association.ts:252` (override), `:274` (super call), plus
  the `NestedAttributesHost` declaration and use in `nested-attributes.ts:619`,
  `:836`.
- The justification comment at `association.ts` `buildRecord`'s
  `initializeAndYield` and its story citation are removed.
- No new `parity:api:calls` / `parity:api:calls:args` baseline rows and no new
  extra surface.
