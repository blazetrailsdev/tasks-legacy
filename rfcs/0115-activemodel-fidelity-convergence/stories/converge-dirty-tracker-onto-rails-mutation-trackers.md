---
title: "Converge the invented DirtyTracker onto Rails' two mutation trackers"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - fan-out-model-dirty-surface-to-dirty-ts
deps-rfc: []
est-loc: 400
pr: null
claim: "2026-08-24T18:15:29Z"
assignee: "converge-dirty-tracker-onto-rails-mutation-trackers"
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/attribute-mutation-tracker.ts` is a faithful,
already-clean port of
`vendor/rails/activemodel/lib/active_model/attribute_mutation_tracker.rb`:
`AttributeMutationTracker` (`:43` ↔ `:7`), `ForcedMutationTracker` (`:150` ↔
`:91`), `NullMutationTracker` (`:207` ↔ `:156`), member for member.

`packages/activemodel/src/dirty.ts:116` defines a **second** tracker,
`DirtyTracker`, and `initInternals` (`dirty.ts:65`) installs it as
`this._dirty`. Its own JSDoc states the deviation:

> Trails consolidates Rails' two mutation trackers into a single
> `DirtyTracker`, so a fresh tracker is the equivalent reset.

Rails' `Dirty` holds **two** instance variables and every reader is a one-line
delegation:

```ruby
def changed?;            mutations_from_database.any_changes?          end  # dirty.rb:286
def changed;             mutations_from_database.changed_attribute_names end # :295
def attribute_was(n);    mutations_from_database.original_value(n.to_s) end  # :305
def changed_attributes;  mutations_from_database.changed_values         end  # :343
```

and `changes_applied` (`:272-278`) is the hand-off the collapse cannot express:

```ruby
mutations_from_database.finalize_changes if @attributes
@mutations_before_last_save = mutations_from_database
@mutations_from_database = nil
```

The collapse is the direct cause of `dirty.ts`'s 102 code lines with no Rails
counterpart: `snapshot` (`:415`), `restore` (`:589`), `_restoreOne` (`:612`),
`reinstateNewRecordChanges` (`:503`), `attributeWritten` (`:533`),
`redetectChanges` (`:570`), `_isInPlaceMutableChange` (`:383`),
`_hasInPlaceMutableChange` (`:389`), `_deleteChange` (`:399`), `_clearChanges`
(`:405`) — plus `DirtyTracker` and `snapshot` and `restore` being three of
`dirty.ts`'s four `novel` names in
`pnpm parity:api:extra --package activemodel`.

`dirty.ts:136` already carries `@noRailsEquivalent CONVERGEABLE — story`, i.e.
this convergence is pre-agreed; it just never got a story.

This is the largest Phase 2 story and the one that unlocks `dirty.rb`'s
3.3x ratio and much of `attribute_set.rb`'s.

## Acceptance criteria

- `Dirty` holds `mutationsFromDatabase` and `mutationsBeforeLastSave` as two
  separate slots, defaulting to `NullMutationTracker`, exactly as
  `dirty.rb:248-253,325-333,372-376` does.
- Every `dirty.ts` reader is the one-line delegation its Ruby counterpart is.
- `changesApplied` performs the `finalize_changes` → reassign → nil sequence of
  `dirty.rb:272-278`.
- `DirtyTracker` is deleted, along with `snapshot`, `restore`, `_restoreOne`,
  `deferNewRecordChanges`, `redetectChanges`, `attributeWritten`,
  `_deriveChanges`, `_deriveWrites`, `_deleteChange`, `_clearChanges`,
  `_isInPlaceMutableChange`, `_hasInPlaceMutableChange` — or each survivor names
  the Rails method it now implements.

  Member list refreshed by #6936, which made the new-record pass lazy:
  `reinstateNewRecordChanges` is now `deferNewRecordChanges` (a queue) drained
  by `_deriveChanges` / `_deriveWrites` against the `Attribute` graph, and
  `attributeWritten` takes only a name. The queue exists solely because this
  tracker records rather than derives, so it dies with it. Its two ActiveRecord
  callers are
  `0023-surfaced-deviations/activerecord-primes-a-new-records-dirty-set`, and
  the transaction-restore caller is
  `0023-surfaced-deviations/transaction-record-state-hand-seeds-the-dirty-tracker`.

- The `@noRailsEquivalent CONVERGEABLE` tag at `dirty.ts:136` is removed, not
  reworded (CLAUDE.md: a deviation-convergence story always converges).
- `pnpm parity:api:extra --package activemodel` shows `dirty.ts` at ≤ 1 novel.
- `activemodel/dirty.json`'s 6 baseline rows shrink; any that converge are
  hand-deleted then `pnpm parity:api:calls:tighten activemodel/dirty.json`.
- Parity deltas non-negative for activemodel **and** activerecord (AR's dirty
  reads through this).

## Verification

```bash
pnpm vitest run packages/activemodel/src/dirty.test.ts packages/activemodel/src/dirty.trails.test.ts packages/activemodel/src/dirty-mutations.test.ts packages/activemodel/src/attribute-mutation-tracker.test.ts
pnpm vitest run packages/activerecord/src/attribute-methods/dirty.test.ts
```
