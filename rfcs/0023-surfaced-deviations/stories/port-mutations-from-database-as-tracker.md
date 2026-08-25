---
title: "mutations_from_database returns the AttributeMutationTracker, not the changes hash"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Surfaced by PR #6619, which ported `attribute_will_change!` to Rails' one-liner
but had to justify the RECEIVER at the call site rather than converge it.

Rails (`vendor/rails/activemodel/lib/active_model/dirty.rb:382-388`):

```ruby
def mutations_from_database
  @mutations_from_database ||= if defined?(@attributes)
    ActiveModel::AttributeMutationTracker.new(@attributes)
  else
    ActiveModel::ForcedMutationTracker.new(self)
  end
end
```

so `mutations_from_database` IS the tracker object, and every Rails caller
reads methods off it (`force_change`, `changed_attribute_names`,
`changed_in_place?`, `finalize_changes`, …). trails has both tracker classes
ported and correct (`packages/activemodel/src/attribute-mutation-tracker.ts`),
but `Model#mutationsFromDatabase` (`packages/activemodel/src/model.ts:2261-2263`)
and `DirtyTracker#mutationsFromDatabase` (`dirty.ts:278-280`) return
`Record<string, [unknown, unknown]>` — the CHANGES hash — instead. The real
tracker is the bespoke `_dirty` (`DirtyTracker`), which is what callers reach
for, so `attributeWillChangeBang` (dirty.ts) carries a comment explaining that
Rails' receiver is `mutations_from_database` while trails' is `_dirty`.

The Record shape is enshrined in tests
(`packages/activemodel/src/dirty-mutations.test.ts`,
`packages/activerecord/src/transactions.trails.test.ts:300,331,353`), which is
why PR #6619 did not flip it.

## Acceptance criteria

- [ ] `mutationsFromDatabase` returns the mutation tracker, memoized, mirroring
      dirty.rb:382-388 including the `defined?(@attributes)` arm that picks
      `AttributeMutationTracker` over `ForcedMutationTracker`.
- [ ] `attributeWillChangeBang` / `Model#attributeWillChange` read
      `this.mutationsFromDatabase.forceChange(attrName)` and the `_dirty`
      receiver comment in `dirty.ts` is deleted.
- [ ] The changes-hash readers those tests pin move to the Rails method that
      actually returns a hash (`changes` / `changes_to_save`), with the test
      names unchanged.
- [ ] activemodel and activerecord dirty-tracking suites green on all three
      lanes.
