---
title: "Cover the disable_joins shapes the deleted DJAS routing gate used to reject"
status: ready
updated: 2026-08-23
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6900 closed `converge-hmt-disable-joins-off-routing-predicate` by deleting
`_canRouteThroughViaDisableJoinsAssociationScope` outright and spelling all
three `find_target` arms the way Rails does:

- `HasManyThroughAssociation#findTarget` → `this.scope().toArray()`
  (`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:228`)
- `SingularAssociation#findTarget` → `this.scope().first()`
  (`.../singular_association.rb:47-53`)
- `HasManyAssociation`'s flat loader dropped its copy of the gate.

`Association#scope`'s `disable_joins` branch now answers every one of them, as
`association.rb:300-306` does.

That closed story's fourth acceptance criterion is **not** met:

> - [ ] A shape the old gate rejected (e.g. polymorphic source without
>       `sourceType`) is covered by a test proving it now routes through DJAS.

The gate used to demand a through reflection, a source reflection, and a
matched polymorphic-source/`sourceType` pairing, falling back to the two-step
`loadHasManyThrough` loader for anything else. Those shapes now go to DJAS
instead, and nothing in the suite exercises the difference —
`disable-joins-routing-widening.test.ts` covers `sourceType` + polymorphic
source, which the old gate **accepted**. The 60 disable-joins assertions that
pass today all describe shapes the gate already allowed.

So the behaviour change the convergence introduced is untested. It is
Rails-correct (`association.rb:300-306` has no gate), but a regression here
would be silent.

## Acceptance criteria

- [ ] A test declares a `disable_joins` through association with a polymorphic
      source and **no** `source_type` — a shape the deleted gate rejected — and
      asserts it loads through DJAS with no JOIN in the emitted SQL.
- [ ] A test covers the remaining rejected shape: a `disable_joins` reflection
      whose `source_type` is set on a non-polymorphic source.
- [ ] Both live in a `*.trails.test.ts` file (no Rails counterpart) and assert
      on emitted SQL rather than on the deleted predicate.
- [ ] Green on SQLite, PostgreSQL and MySQL/MariaDB.
