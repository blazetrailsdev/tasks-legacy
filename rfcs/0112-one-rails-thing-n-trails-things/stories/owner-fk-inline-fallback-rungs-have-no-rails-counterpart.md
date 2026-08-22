---
title: "Remove the no-reflection fallback rungs from ownerForeignKeyColumns"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: dead-mixin-companions
deps: []
deps-rfc: []
est-loc: 80
pr: 6738
claim: "2026-08-19T12:59:52Z"
assignee: "days-into-week-duplicated-in-date-calculations"
blocked-by: null
closed-reason: null
---

## Context

`ownerForeignKeyColumns`
(`packages/activerecord/src/associations/foreign-association.ts`, added in
PR 5636) is the single owner-FK derivation site. Its first two rungs mirror Rails
— `options.foreignKey`, then the rich reflection's `foreignKey`
(`reflection.rb` `compute_foreign_key`, keyed on `reflection.active_record`).

Everything below those two rungs is a trails invention with no Rails
counterpart:

```ts
if (options.as) return [`${underscore(options.as)}_id`];
if (options.queryConstraints) return options.queryConstraints;
const pk = options.primaryKey ?? ctor.primaryKey ?? "id";
if (Array.isArray(pk)) return pk.map((col) => `${underscore(ctor.name)}_${col}`);
return [`${underscore(ctor.name)}_id`];
```

In Rails an association always has a registered reflection, so
`reflection.foreign_key` never has a fallback to fall through to. These rungs
exist only because trails' free-function engine path (`findTarget` in
`has-many-association.ts`, `_findHasOneTarget` in `singular-association.ts`)
can be reached for an association with no resolvable reflection.

They also under-approximate the reflection: the `as:` rung skips
`derive_fk_query_constraints` widening, and the CPK rung reimplements a
narrowed `deriveForeignKey`. Sibling stories
`loadhasmany-no-reflection-fallback-*` (0023, both done) removed the analogous
fallbacks elsewhere.

## Acceptance criteria

- Establish whether any live call path reaches `ownerForeignKeyColumns` without
  a resolvable reflection (instrument or assert in the AR suite).
- If none does, delete the rungs below the reflection and let the helper be
  `options.foreignKey ?? reflection.foreignKey`, matching Rails.
- If some path does, that path is the deviation — register it rather than
  keeping the ladder, and narrow the rungs to that one caller.
- No test renames; no new Rails-named tests.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
