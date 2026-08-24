---
title: "Parked assignAttributes promises never drain on models without acceptsNestedAttributesFor"
status: draft
updated: 2026-08-08
rfc: "0087-awaitable-association-writers-only"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `_assignAttribute` in PR #6220.

`assignAttributes` (`packages/activerecord/src/persistence.ts`) answers a promise
when a send owed DB I/O — `replace` (`collection_association.rb:46-48`),
`ids_writer` (`:61-83`), has_one's displacing writer
(`has_one_association.rb:59-84`). Three callers cannot await it and park it with
`parkNestedReaderLoad` instead:

- `_applyScopeAttributes` (`base.ts`) and `populateWithCurrentScopeAttributes`
  (`scoping.ts`) — `populate_with_current_scope_attributes` runs from
  `initialize` (`scoping.rb:60-66`, `core.rb:474`), and a JS constructor cannot
  await. Reached by any `Model.where(assoc: x).new` / `.create`.
- the `attributes=` accessor (`base.ts`) — `alias attributes= assign_attributes`
  (`activemodel/attribute_assignment.rb:36`); a TS `set` accessor cannot await.
- `Association#buildRecord`'s `initializeAndYield`
  (`packages/activerecord/src/associations/association.ts`, added in PR #7005) —
  `build_record` yields and returns synchronously
  (`associations/association.rb:383-388`) and `CollectionAssociation#build`
  returns the record itself (`collection_association.rb:117-122`), so the
  deferred `initialize_attributes` assign is parked. Reached by any
  `owner.association(name).build(...)` whose `scope_for_create` names an
  association writer (`relation.rb:1231-1235`).

**The park has no drain on most models.** `parkNestedReaderLoad` queues onto
`_pendingNestedReaderLoads`, and the only call to
`awaitPendingNestedReaderLoads` is `nested-attributes.ts:182` — inside the
`save` wrapper that `acceptsNestedAttributesFor` installs on
`modelClass.prototype`. A model that never calls `acceptsNestedAttributesFor`
has no wrapper, so a promise parked on it is never awaited and its rejection
never rethrown: the write may still be in flight when `save` returns, and an
error in it is swallowed (`parkNestedReaderLoad` deliberately captures the
rejection so it cannot surface as an unhandled rejection).

Rails has no equivalent hole — `assign_attributes` completes inline, so
`Firm.where(firm: f).new` has done the write before `new` returns.

The two parks are correct as deferrals; what is missing is that the drain is
conditional on an unrelated macro.

## Converged shape

The drain belongs where every model reaches it, not on the nested-attributes
save wrapper: `save` / `save!` (and the existing `create` / `create!` drain)
should await `awaitPendingNestedReaderLoads` for any record with parked work,
so a parked promise's rejection reaches the caller and its write is ordered
before the INSERT/UPDATE. `acceptsNestedAttributesFor`'s wrapper then keeps only
`processNestedAttributes`.

Better still, and the real endgame: neither park exists once the two callers can
await. `attributes=` already has the awaitable spelling (`setAttributes`), so the
accessor could raise for a key whose writer owes I/O rather than park. The
constructor arm (`scoping.rb:60-66` from `core.rb:474`) is the one genuine
language limit and needs the drain either way.

## Acceptance criteria

- [ ] A promise parked by `_applyScopeAttributes` /
      `populateWithCurrentScopeAttributes` / `attributes=` is awaited by
      `save` / `save!` on a model that never calls
      `acceptsNestedAttributesFor`.
- [ ] A rejection in a parked assignment surfaces out of `save`, rather than
      being swallowed.
- [ ] Regression test that fails on the current shape: a model with **no**
      `acceptsNestedAttributesFor`, an association-valued scope attribute
      (`Model.where(assoc: x).new`), asserting the write is complete after
      `save` — and a second asserting a failing parked write rejects `save`.
- [ ] `awaitPendingNestedReaderLoads` has no caller that depends on
      `acceptsNestedAttributesFor` having run.
