---
title: "unknown-primary-key-receives-model-not-relation"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down RFC 0096 wave-2 naming rows.

`TokenFor#find_by_token_for`
(`vendor/rails/activerecord/lib/active_record/token_for.rb:41-44`) raises
`UnknownPrimaryKey.new(self)` — **the relation**, not its model. Rails'
`UnknownPrimaryKey#initialize` (`errors.rb:475-484`) then renders
`"Unknown primary key for table #{model.table_name} in model #{model}."`, so
the message interpolates the relation's `to_s`, and the error's `model` reader
holds the relation.

trails (`packages/activerecord/src/relation.ts:6403`) raises
`new UnknownPrimaryKey(this.model)`, so both the message's model half and the
error's `model` reader carry the model class instead. `UnknownPrimaryKey`'s
constructor is typed `typeof Base | null`
(`packages/activerecord/src/errors.ts:800-814`), which is what forces the
substitution — the type has to widen before the call site can converge.

RFC 0096 leaves the row (`new`: Ruby `ref:this` → TS `ref:model`) standing
because converging it changes the error message text, which is outside a
naming story.

## Acceptance criteria

- [ ] `UnknownPrimaryKey` accepts whatever `find_by_token_for` passes in Rails,
      and `findByTokenFor` passes `this`.
- [ ] The rendered message matches Rails for both the relation and the
      model-class call sites (`errors.rb:477`), including the `#{model}` half.
- [ ] A test asserts the message for a relation with no primary key; it fails
      on baseline.
- [ ] `pnpm parity:api:calls:args` stays green and the `relation.ts`
      `find_by_token_for` naming row is gone.
