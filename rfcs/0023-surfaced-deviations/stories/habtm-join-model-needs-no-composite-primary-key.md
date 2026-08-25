---
title: "HABTM join model declares a composite primary key Rails does not"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' HABTM join model
(`activerecord/lib/active_record/associations/builder/has_and_belongs_to_many.rb:13-56`)
declares no primary key: a join table has no id column, and the cleanup Rails
issues is `association(:#{middle}).delete_all(:delete_all)`
(`associations.rb:1888`), which builds its own WHERE from the has_many scope.

trails gives the join model a composite primary key of the two join keys —
after PR #6836 that lives at
`packages/activerecord/src/associations/builder/has-and-belongs-to-many.ts`
(`joinModel.primaryKey = [leftReflection.foreignKey, rightReflection.foreignKey]`),
having moved off the deleted `createHabtmJoinModel`. It exists because trails'
delete/destroy path issues PK-based WHERE clauses where Rails does not, and it
has a measured cost: a composite PK on the join model forces the eager-load
bypass (see `habtm-join-model-composite-pk-forces-eager-bypass`).

The `dependent: "delete"` on the middle reflection is the other half of the same
stand-in.

## Converged shape

The middle association's cleanup goes through the `delete_all(:delete_all)`
scope Rails uses, so the join model needs no primary key and
`middleOptions.dependent` goes away with it. That is the same convergence
`converge-habtm-builder-to-rails-macro-sequence` approaches from the macro side
— check it before starting so the two do not overlap.

## Acceptance criteria

- [ ] The join model carries no `primaryKey` assignment.
- [ ] `destroyAssociations` deletes join rows through the middle association's
      scope, not a PK-based WHERE.
- [ ] The HABTM destroy and eager-load tests stay green, and the eager-load
      bypass this PK forced is no longer needed.
