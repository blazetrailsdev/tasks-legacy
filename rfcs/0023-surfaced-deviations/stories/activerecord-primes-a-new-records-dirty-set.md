---
title: "ActiveRecord primes a new record's dirty set where Rails derives it on read"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
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

Surfaced closing `compute-record-dirtiness-lazily` (#6936).

Rails computes a new record's dirtiness when a read asks for it:
`ActiveModel::Dirty#changed?` delegates to
`mutations_from_database.any_changes?`
(`vendor/rails/activemodel/lib/active_model/dirty.rb:286`), which walks the
`Attribute` graph via `Attribute#changed?`
(`vendor/rails/activemodel/lib/active_model/attribute.rb:139-141`). Nothing in
`ActiveRecord::Core#initialize` or `Persistence#_create_record` primes a dirty
set — there is no set to prime.

trails still primes one. #6936 made the priming lazy (a queue drained on the
first read) but did not remove it, so two ActiveRecord call sites remain with no
Rails counterpart:

- `_reinstateConstructorDirtiness` (`packages/activerecord/src/base.ts`), run
  from the constructor's `!wasSuppressed` branches, calls
  `DirtyTracker#deferNewRecordChanges(record._attributes)`.
- `_createRecord` (`packages/activerecord/src/persistence.ts`) calls
  `this._dirty.deferNewRecordChanges(this._attributes, _pkSet)` before the
  INSERT so `attribute_names_for_partial_inserts` sees a populated changed set,
  where Rails' `changed_attribute_names_to_save`
  (`vendor/rails/activerecord/lib/active_record/attribute_methods/dirty.rb`)
  derives it on the spot.

Both exist only because `DirtyTracker` records changes rather than deriving
them from the `AttributeSet`. They are the ActiveRecord half of that
convergence: `converge-dirty-tracker-onto-rails-mutation-trackers` (RFC 0115)
deletes the tracker in `activemodel`, and these two callers have to go with it.

## Converged shape

`Model.new(attrs)` and `_create_record` prime nothing. `changed?`,
`changed_attribute_names_to_save` and the partial-insert column selection each
ask the `Attribute` graph when they run, so a record built by assignment is
dirty because its attributes are `FromUser`-over-default — not because a
constructor pass said so. `deferNewRecordChanges`, `_deriveChanges`,
`_deriveWrites`, `_pendingNames` and `_pendingWrites` all disappear with it.

## Acceptance criteria

- `_reinstateConstructorDirtiness` is deleted, and no constructor branch primes
  a dirty set.
- `_createRecord` no longer primes one; partial inserts still select the
  columns Rails selects (`dirty_test.rb` / `persistence_test.rb` partial-insert
  cases stay green on all four adapter lanes).
- `DirtyTracker#deferNewRecordChanges` and the pending-derivation queue are gone
  from `packages/activemodel/src/dirty.ts`.
