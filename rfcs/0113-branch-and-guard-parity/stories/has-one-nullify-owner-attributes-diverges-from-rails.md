---
title: "has_one nullifyOwnerAttributes misses the primary-key guard and nulls the polymorphic type column"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
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

`HasOneAssociation#nullifyOwnerAttributes`
(`packages/activerecord/src/associations/has-one-association.ts`) sources its
column list from the module-level `nullifiedOwnerAttributes(assoc)` helper,
which delegates to `ForeignAssociation.nullifiedOwnerAttributes` and writes
`null` for every column it returns.

Rails has TWO distinct methods and we collapsed them onto the wrong one:

- `ForeignAssociation#nullified_owner_attributes`
  (`vendor/rails/activerecord/lib/active_record/associations/foreign_association.rb:13-18`)
  — foreign key columns PLUS `reflection.type`. Used by has_many's nullify.
  Our `ForeignAssociation.nullifiedOwnerAttributes` mirrors this exactly.
- `HasOneAssociation#nullify_owner_attributes`
  (`vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb:118-122`):

  ```ruby
  def nullify_owner_attributes(record)
    Array(reflection.foreign_key).each do |foreign_key_column|
      record[foreign_key_column] = nil unless foreign_key_column.in?(Array(record.class.primary_key))
    end
  end
  ```

Two divergences follow, both on the has_one `remove_target!` nullify arm
(has_one_association.rb:104-106) and on `replace`'s failed-save rollback (:76):

1. **Missing primary-key guard.** Rails refuses to null a foreign key column
   that is part of the record's own primary key; we null it unconditionally. A
   has_one whose child keys itself by the owner FK (or a composite PK
   overlapping the FK) has its primary key blanked in memory.
2. **Nulls the polymorphic type column.** Rails' has_one version iterates only
   `reflection.foreign_key` and never touches `reflection.type`; ours adds the
   `as:` type column because it reuses the ForeignAssociation helper.

Surfaced while reading `remove_target!` for
`eliminate-pending-removal-target-state` (PR #5663). Not covered by any current
test — the canonical has_one fixtures have no FK/PK overlap.

## Acceptance criteria

- [ ] `HasOneAssociation#nullifyOwnerAttributes` iterates
      `reflection.foreignKey` only, and skips any column present in
      `Array(record.constructor.primaryKey)`.
- [ ] The `ForeignAssociation.nullifiedOwnerAttributes` helper is left alone —
      it is the faithful port of the has_many-side method and keeps its callers.
- [ ] A test covers a has_one whose foreign key overlaps the child's primary
      key: `remove_target!`'s nullify arm leaves the primary key intact.
- [ ] A test covers a polymorphic `as:` has_one: the nullify arm does not blank
      the `*_type` column.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
