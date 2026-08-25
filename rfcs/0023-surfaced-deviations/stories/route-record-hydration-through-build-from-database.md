---
title: "Route record hydration through Builder#build_from_database so LazyAttributeSet is actually used"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Route record hydration through `Builder#build_from_database` so `LazyAttributeSet` is actually used

## Context

Surfaced in PR #6585, which ported `LazyAttributeSet` / `LazyAttributeHash` at
their Rails names and made `Builder#buildFromDatabase` return
`new LazyAttributeSet(values, types, additionalTypes, defaultAttributes)`
(`vendor/rails/activemodel/lib/active_model/attribute_set/builder.rb:15-17`).

The class is correct but unreachable: **nothing in the tree calls
`buildFromDatabase` outside its own tests.**

    grep -rn "buildFromDatabase\|new LazyAttributeSet" --include=*.ts packages/

returns only `packages/activemodel/src/attribute-set/builder.ts` and its specs.
`Model.attributesBuilder` exists (`packages/activerecord/src/model-schema.ts:748`)
and memoizes an `AttributeSet::Builder`, but no caller ever invokes
`.buildFromDatabase()` on it.

In Rails, `build_from_database` IS the hydration path — `ActiveRecord::Core#init_with_attributes`
and `Persistence::ClassMethods#instantiate` reach it through
`attributes_builder.build_from_database(attributes, column_types)`
(`vendor/rails/activerecord/lib/active_record/persistence.rb`, `core.rb`). trails
instead populates an eager `AttributeSet` by another route and then narrows it
with the trails-only `narrowTo` (`packages/activerecord/src/inheritance.ts:597`,
`narrowToProjectedColumns`).

So the laziness that makes loading a wide row cheap — materializing each
`Attribute` only on first read — is ported but never exercised.

## Converged shape

`instantiate` / `init_with_attributes` build their attribute set through
`attributesBuilder().buildFromDatabase(values, columnTypes)`, so a loaded record
carries a `LazyAttributeSet` and unselected columns resolve through
`LazyAttributeSet#default_attribute`'s uninitialized arm on first read
(`builder.rb:80-84`) instead of being force-written by `narrowTo`.

Note two things that must move with it:

- `narrowTo`, `snapshotValues` and `forgetAssignmentsBang`
  (`packages/activemodel/src/attribute-set.ts`) are trails-only and read raw
  `_attributes` deliberately — routing them through the materializing
  `attributes()` seam would force the whole row on every load. Once hydration is
  lazy, `narrowToProjectedColumns` is likely redundant outright rather than
  merely inert, and should be deleted rather than adapted.
- Rails' own `column_types` slice is the `additional_types` argument, which is
  what `LazyAttributeSet` already threads.

## Acceptance criteria

- [ ] Record hydration constructs its attribute set via
      `attributesBuilder().buildFromDatabase(...)`.
- [ ] A loaded record's `_attributes` is a `LazyAttributeSet`, and an
      unread column is not materialized until first read (assert on the
      internal map, as `builder.test.ts` already does).
- [ ] `narrowToProjectedColumns` is deleted, or its remaining job is stated
      and covered.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
