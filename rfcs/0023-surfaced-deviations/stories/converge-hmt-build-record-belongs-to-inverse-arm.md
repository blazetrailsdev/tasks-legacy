---
title: "HasManyThroughAssociation#build_record drops the non-Rails belongs_to inverse arm"
status: draft
updated: 2026-08-12
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

Surfaced while landing #6388 (`converge-collection-proxy-build-record`), which
moved the through-build wiring out of `CollectionProxy#_buildThrough` and into
`HasManyThroughAssociation#buildRecord`
(`packages/activerecord/src/associations/has-many-through-association.ts:300`).

Rails' `HasManyThroughAssociation#build_record`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:89-111`)
wires the pre-built join row onto **only two** inverse shapes:

```ruby
if inverse
  if inverse.collection?
    record.send(inverse.name) << build_through_record(record)
  elsif inverse.has_one?
    record.send("#{inverse.name}=", build_through_record(record))
  end
end
```

trails carries a third arm — a `belongs_to` source inverse gets the join row
pushed through `inverseAssoc.writer(...)`. It is pre-existing debt (it lived in
`CollectionProxy#_buildThrough` before #6388 relocated it verbatim) and is
annotated as such at the call site, but it is a real divergence: in Rails the
foreign key for that shape arrives from the through scope inside
`initialize_attributes`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:224`,
`scope_for_create`), not from an extra writer call.

Three Rails tests currently depend on the arm — remove it without fixing the
through scope and they red:

- `HasManyThroughAssociationsTest > build for has many through association`
- `HasManyThroughAssociationsTest > cpk association build through singular`
- `HasManyThroughAssociationsTest > through record is built when created with where`

(`packages/activerecord/src/associations/has-many-through-associations.test.ts:2168`,
`:2525`, `:1220`.)

## Converged shape

`buildRecord`'s inverse block matches Rails exactly — a `collection?` arm and a
`has_one?` arm, nothing else. The `belongs_to` FK comes from
`throughScopeAttributes()` / `scope_for_create` flowing through
`Association#initializeAttributes`, so the three tests above stay green with the
writer arm deleted.

## Acceptance criteria

- [ ] The `inverseAssoc.writer(...)` arm is deleted from
      `HasManyThroughAssociation#buildRecord`; the remaining branches are the
      `collection?` / `has_one?` pair Rails has, in Rails' order.
- [ ] The three tests named above pass without it, because the through scope
      supplies the source foreign key.
- [ ] `pnpm parity:api:calls` / `pnpm parity:api:calls:args` green.
