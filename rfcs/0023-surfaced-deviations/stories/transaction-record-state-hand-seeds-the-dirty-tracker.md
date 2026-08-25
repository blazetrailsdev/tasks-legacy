---
title: "Transaction record state hand-seeds the dirty tracker where Rails nils it"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced closing `route-ar-dup-attribute-duplication-through-the-initialize-dup-links`
and `compute-record-dirtiness-lazily` (#6936).

Rails restores a rolled-back record's dirty state by dropping both mutation
trackers and letting them rebuild from `@attributes`:

```ruby
# vendor/rails/activerecord/lib/active_record/transactions.rb:369-395
@attributes = @attributes.map { |attr| ... }
@mutations_from_database = nil
@mutations_before_last_save = nil
```

trails' `restoreTransactionRecordState`
(`packages/activerecord/src/transactions.ts`) nils the two fields Rails nils and
then has to seed the surviving `DirtyTracker` by hand:

```ts
r._dirty.snapshot(restoreState.attributes);
r._dirty.clearChangesInformation();
r._dirty.redetectChanges(r._attributes);
```

None of those three has a Rails counterpart, and the sequence is
order-dependent in a way Rails' is not (the in-code comment already notes the
primary key must be restored before `redetectChanges` runs, because it only
sets entries and never deletes them). #6936 had to add an `_attrs` rebind
inside `redetectChanges` on top, because `snapshot(preTxAttrs)` leaves the
tracker bound to the pre-TX `AttributeSet` while every later read and
derivation must ask the live one — a binding Rails gets for free by rebuilding
the tracker from `@attributes`.

`rememberTransactionRecordState` has the mirror-image problem: it reads
`r._dirty.changes` and replays each `[original]` back into the snapshot with
`writeFromUser`, where Rails just deep-dups `@attributes` and lets the
`Attribute`s carry their own originals.

## Converged shape

Both halves stop talking to a tracker. `remember_transaction_record_state`
snapshots `@attributes` alone; `restore_transaction_record_state` maps the
attributes back and nils the two tracker slots, which rebuild from the restored
`@attributes` on the next ask. `snapshot`, `redetectChanges` and
`clearChangesInformation` drop out of the AR call graph.

Depends on `converge-dirty-tracker-onto-rails-mutation-trackers` (RFC 0115),
which supplies the two rebuildable slots.

## Acceptance criteria

- `restoreTransactionRecordState` performs the `attributes.map` + two nils of
  `transactions.rb:369-395` and calls no tracker seeding method.
- `rememberTransactionRecordState` no longer reads `changes` or replays
  originals through `writeFromUser`.
- `transactions.test.ts` stays green on all four adapter lanes — in particular
  `rollback dirty changes then retry save on new record with autosave
association`, which is the case the current seeding order exists for.
