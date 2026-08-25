---
title: "BelongsToAssociation#primary_key is missing; port carries an invented associationPrimaryKeys"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:api --package activerecord` reports 2 missing methods; one is
`ActiveRecord::Associations::BelongsToAssociation#primary_key`
(`vendor/rails/activerecord/lib/active_record/associations/belongs_to_association.rb:151-153`):

```ruby
def primary_key(klass)
  reflection.association_primary_key(klass)
end
```

It is called at `:120` (`klass.unscoped.where!(primary_key(klass) => foreign_key)`),
`:135` and `:143`. The port has no `primaryKey(klass)`; it carries an invented
plural helper `associationPrimaryKeys(klass)`
(`packages/activerecord/src/associations/belongs-to-association.ts:352`) with a
hand-rolled composite/`id`-inference body, called from `:194` and `:400`.

## Converged shape

`primaryKey(klass)` on `BelongsToAssociation`, one line, delegating to
`this.reflection.associationPrimaryKey(klass)`, with the three Rails call sites
routed through it. The composite-PK inference in `associationPrimaryKeys` belongs
in `AssociationReflection#associationPrimaryKey` if it is needed at all — Rails
does not do it here.

## Acceptance criteria

- `belongs-to-association.ts` declares `primaryKey(klass)` mirroring
  `belongs_to_association.rb:151-153`; `associationPrimaryKeys` is gone or
  reduced to what Rails actually has.
- `pnpm parity:api --package activerecord` loses the `primary_key` missing row.
- AR association tests green.
