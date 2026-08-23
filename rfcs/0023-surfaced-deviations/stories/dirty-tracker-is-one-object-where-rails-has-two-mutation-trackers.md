---
title: "dirty-tracker-is-one-object-where-rails-has-two-mutation-trackers"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `DirtyTracker` is one object where Rails has two `AttributeMutationTracker` instances

## Context

Rails' `ActiveModel::Dirty` holds TWO trackers — `mutations_from_database` and
`mutations_before_last_save` (`activemodel/lib/active_model/dirty.rb:274-284`) —
each an `AttributeMutationTracker`. Every dirty reader is
`<one of them>.changed?(attr_name.to_s, **options)`:
`attribute_changed?` (dirty.rb:300-302), `attribute_previously_changed?`
(:310-312), and ActiveRecord's `saved_change_to_attribute?` /
`will_save_change_to_attribute?`
(`activerecord/lib/active_record/attribute_methods/dirty.rb:86-88`, `:138-140`).
The receiver IS the choice of change set; `changed?` itself takes only
`attr_name` and `**options` (`attribute_mutation_tracker.rb:44-48`).

trails has a single `DirtyTracker` (`packages/activemodel/src/dirty.ts`) holding
both change sets in `_changedAttributes` / `_previousChanges`, so the set Ruby
names by receiver has to travel as `DirtyTracker#isChanged`'s LEADING argument.
PR #6893 consolidated the four readers onto that one body and tagged the
argument-shape divergence `@missingRailsArgs changed? — CONVERGEABLE` at three
call sites (`activemodel/src/dirty.ts`,
`activerecord/src/attribute-methods/dirty.ts` x2).

A faithful `AttributeMutationTracker` port already exists and is unused by
`DirtyTracker`: `packages/activemodel/src/attribute-mutation-tracker.ts`, with
`isChanged(attrName, options)`, `changeToAttribute`, `originalValue`,
`anyChanges`, `changedAttributeNames`, plus `ForcedMutationTracker` and
`NullMutationTracker`. Wiring `DirtyTracker` to hold two instances of it retires
the leading argument and the three tags with it.

## Acceptance criteria

- [ ] `DirtyTracker` exposes `mutationsFromDatabase` /
      `mutationsBeforeLastSave` as `AttributeMutationTracker` instances
      (dirty.rb:274-284), not as plain change-set records.
- [ ] `DirtyTracker#isChanged`'s leading `changes` argument is gone; each reader
      calls `<tracker>.isChanged(attrName, options)` with Rails' argument list.
- [ ] The three `@missingRailsArgs changed?` tags are deleted, and
      `pnpm parity:api:calls:args` stays green with no new rows.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
