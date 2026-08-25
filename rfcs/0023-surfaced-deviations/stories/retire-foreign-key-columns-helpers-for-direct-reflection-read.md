---
title: "Retire the three foreignKeyColumns() copies for a direct reflection.foreign_key read"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Retire the three `foreignKeyColumns()` copies for a direct `reflection.foreign_key` read

## Context

Surfaced while converging `HasOneAssociation#nullifyOwnerAttributes` in PR #6815
(story `has-one-nullify-owner-attributes-converge`).

Rails never wraps the owner foreign key in a helper. Every site reads
`reflection.foreign_key` inline:

- `vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb:119-123`
  — `Array(reflection.foreign_key).each { |fkc| ... }`
- `vendor/rails/activerecord/lib/active_record/associations/foreign_association.rb:13-18`
  — `Array(reflection.foreign_key).each { |fk| attrs[fk] = nil }`

trails instead carries a private `foreignKeyColumns(): string[]` on three
classes, each delegating to the shared `ownerForeignKeyColumns`
(`packages/activerecord/src/associations/foreign-association.ts:17-34`):

- `packages/activerecord/src/associations/has-one-association.ts:548`
  (plus the singular `foreignKeyColumn()` at `:557`)
- `packages/activerecord/src/associations/collection-association.ts:1094`
  (plus `foreignKeyColumn()` at `:1103`)
- reached structurally as `(assoc as any).foreignKeyColumns()` from
  `has-many-association.ts:546` and `has-one-association.ts:714,779`

`ownerForeignKeyColumns` exists to spell `options[:foreign_key] ||
reflection.foreign_key`. That first rung is **redundant**: the rich reflection's
own `foreignKey` getter (`packages/activerecord/src/reflection.ts:795-817`)
already checks `this.options.foreignKey` first, then
`this.options.queryConstraints`, then derives. So the helper resolves the same
value the reflection getter does, one indirection later and under a name the
call-set ratchet cannot match to Ruby's `foreign_key`.

That last part is not theoretical: it kept a `foreign_key` row alive in
`scripts/api-compare/call-mismatches-exclude/activerecord/associations/has-one-association.json`
for `nullify_owner_attributes` until PR #6815 read the reflection directly, at
which point the row retired with no behaviour change. The same mechanism is
holding rows on the other two classes.

## Converged shape

Read the reflection in the body, as Rails does:

```ts
const reflection = _reflectOnAssociation(this.owner.constructor as typeof Base, this.reflection.name);
for (const foreignKeyColumn of arrayWrap(reflection?.foreignKey)) { ... }
```

`_reflectOnAssociation` is already exported from
`packages/activerecord/src/reflection.ts:2290`.

Keep `ownerForeignKeyColumns` only if a caller genuinely has an ad-hoc
association definition with no registered class-level reflection (a through
step, a HABTM synthesised holder) — that is the one case the options rung
answers and the reflection getter cannot. Establish whether such a caller exists
before deleting it; if none does, delete it and its `NoMethodError` fallback too.

## Acceptance criteria

- [ ] `foreignKeyColumns()` / `foreignKeyColumn()` are removed from
      `has-one-association.ts` and `collection-association.ts`, or reduced to the
      genuinely-ad-hoc case established above, with the Rails cite at the call site.
- [ ] The structural `(assoc as any).foreignKeyColumns()` reaches in
      `has-many-association.ts:546` and `has-one-association.ts:714,779` are
      updated with them.
- [ ] Any `foreign_key` call-set row that retires as a result is deleted by hand
      via `serializeBaseline` and the shard mark tightened with
      `pnpm parity:api:calls:tighten <shard>`. No `--write`, no reseed.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no new
      `parity:api:extra` surface.
- [ ] `dependent: :nullify` and owner-FK coverage still passes (has-one,
      has-many, has-many-through, polymorphic, STI-owner, composite-PK).
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
