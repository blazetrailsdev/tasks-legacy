---
title: "Port set_owner_attributes from ForeignAssociation instead of the has_one-only hand-rolled zip"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while reviewing PR #6367 (`converge-association-scope-pass-self`),
which touched the FK-source line in `HasOneAssociation#setOwnerAttributes` but
left the surrounding body's pre-existing drift alone.

Rails puts `set_owner_attributes` on the `ForeignAssociation` module
(`vendor/rails/activerecord/lib/active_record/associations/foreign_association.rb:20-38`),
included by both `HasManyAssociation` and `HasOneAssociation`:

```ruby
def set_owner_attributes(record)
  return if options[:through]

  primary_key_attribute_names = Array(reflection.join_primary_key)
  foreign_key_attribute_names = Array(reflection.join_foreign_key)

  primary_key_foreign_key_pairs = primary_key_attribute_names.zip(foreign_key_attribute_names)

  primary_key_foreign_key_pairs.each do |primary_key, foreign_key|
    value = owner._read_attribute(foreign_key)
    record._write_attribute(primary_key, value)
  end

  if reflection.type
    record._write_attribute(reflection.type, owner.class.polymorphic_name)
  end
end
```

For a has_one / has_many reflection `join_primary_key` is `foreign_key` (the
target's FK column) and `join_foreign_key` is `active_record_primary_key` (the
owner's PK) — `reflection.rb:606-616`. So despite the block's parameter names,
Rails reads the OWNER's primary key and writes the RECORD's foreign key.

trails' body lives on `HasOneAssociation` only
(`packages/activerecord/src/associations/has-one-association.ts:526-559`) and
picks the same two columns by a different route, so the _values_ line up but
the derivation does not:

- No `return if options[:through]` guard.
- Owner column is `reflection.options.primaryKey ?? ctor.primaryKey ?? "id"`
  rather than `reflection.joinForeignKey` (= `active_record_primary_key`), so a
  reflection-level `active_record_primary_key` override is not honoured.
- Target column comes from the local `foreignKeyColumn()` helper /
  `reflection.foreignKey` rather than `reflection.joinPrimaryKey`.
- The pairing is a manual index loop with a `pks[i] ?? pks[0]` fallback rather
  than Rails' `zip`, so a length mismatch silently reuses the first PK instead
  of pairing against `nil`.
- The polymorphic type write is gated on `options.as` and derives the column as
  `options.foreignType ?? "#{as}_type"`, where Rails gates on `if reflection.type`
  and writes `reflection.type`.
- `HasManyAssociation` gets none of it — Rails shares one body via the module.

`AssociationReflection#joinPrimaryKey` / `#joinForeignKey` already exist in
`packages/activerecord/src/reflection.ts:1106-1120`, and PR #6367 made
`Association#reflection` the rich reflection, so both are readable off
`this.reflection` with no lookup.

## Acceptance criteria

- [ ] `setOwnerAttributes` is ported as Rails writes it — the `options[:through]`
      early return, `Array(joinPrimaryKey).zip(Array(joinForeignKey))`, the
      `owner._readAttribute(foreignKey)` → `record._writeAttribute(primaryKey, …)`
      loop, and the `if (reflection.type)` polymorphic write.
- [ ] It lives in a `foreign-association.ts` `ForeignAssociation` mixin mirroring
      `foreign_association.rb`, wired into both `HasOneAssociation` and
      `HasManyAssociation` via the settled `this`-typed-function idiom, rather
      than being duplicated or left has_one-only.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; any row the
      convergence retires is deleted by hand (only-shrink, no `--write`).
- [ ] AR suites pass on all three adapter lanes.
