---
title: "remember_transaction_record_state deep-dups and dirty-reverts the snapshot where Rails aliases @attributes"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced in PR #6752, which converged
`restore_transaction_record_state` onto the Rails body
(`vendor/rails/activerecord/lib/active_record/transactions.rb:467-491`). Its
partner, `remember_transaction_record_state`, is still a deviation.

Rails stores the **live** `@attributes` object in the snapshot
(`vendor/rails/activerecord/lib/active_record/transactions.rb:440-450`):

```ruby
def remember_transaction_record_state
  @_start_transaction_state ||= {
    id: id,
    new_record: @new_record,
    previously_new_record: @previously_new_record,
    destroyed: @destroyed,
    attributes: @attributes,
    frozen?: frozen?,
    level: 0
  }
  @_start_transaction_state[:level] += 1
  ...
end
```

`attributes: @attributes` is a plain reference. It is safe because each
`Attribute` is immutable-on-write: `write_from_user` replaces the entry rather
than mutating it, so the snapshotted set keeps the pre-TX `Attribute` objects,
and each one still carries its own `original_attribute`. That is exactly what
makes `restore_transaction_record_state`'s `map` +
`with_value_from_user` reconstruction work.

trails' `rememberTransactionRecordState`
(`packages/activerecord/src/transactions.ts`) instead does:

```ts
const snapshotAttrs = r._attributes.deepDup();
const dirtyChanges = r._dirty.changes as Record<string, [unknown, unknown]>;
for (const [name, [original]] of Object.entries(dirtyChanges)) {
  snapshotAttrs.writeFromUser(name, original);
}
```

Two deviations stacked in four lines:

1. **`deepDup()` where Rails aliases.** Rails takes no copy.
2. **The dirty-revert loop has no Rails counterpart at all.** It walks
   `_dirty.changes` and writes each attribute's pre-change value back into the
   snapshot, to synthesize a "DB baseline" that Rails gets for free from each
   `Attribute`'s `original_attribute`.

Both exist for the same root cause: trails' dirty state lives in an external
`DirtyTracker` keyed on a snapshot `Map`, not on the `Attribute` objects — so the
snapshot cannot carry its own baseline and one has to be reconstructed. That root
cause is already tracked as
`0023-surfaced-deviations/construction-time-dirty-baseline-hides-ctor-assignments`
(cited in `DirtyTracker#deepDup`'s own `@noRailsEquivalent`), whose fix — making
`DirtyTracker` derive from the `AttributeSet` — retires this too.

Consequence worth naming: because the snapshot is a reverted deep copy rather
than the live set, `restore_transaction_record_state`'s `map` cannot be the whole
restore. PR #6752 ported the `map` faithfully but still has to follow it with
`_dirty.snapshot(...)` / `clearChangesInformation()` / `redetectChanges(...)` to
move the changed-set, and that triple is the visible residue of this story.

## Converged shape

`rememberTransactionRecordState` stores `r._attributes` directly, with no
`deepDup()` and no dirty-revert loop, matching `transactions.rb:445`. That
requires the dirty baseline to live on the `Attribute` objects, so this story
lands **after** (or as part of) the `DirtyTracker`-derives-from-`AttributeSet`
work; once it does, the three `_dirty.*` calls at the tail of
`restoreTransactionRecordState` are deleted too and the Rails `map` alone
restores the record.

Do not converge the `deepDup()` on its own — aliasing the live set while the
tracker still keys on a `Map` would leave the snapshot with no baseline at all.

## Acceptance criteria

- [ ] `rememberTransactionRecordState` mirrors `transactions.rb:440-450`:
      `attributes: @attributes` with no copy and no dirty-revert loop.
- [ ] The `_dirty.snapshot` / `clearChangesInformation` / `redetectChanges`
      triple at the end of `restoreTransactionRecordState` is deleted; the
      Rails `map` is the whole attribute restore.
- [ ] `transactions.test.ts` rollback-dirty coverage
      (`rollback dirty changes`, `rollback dirty changes multiple saves`,
      `rollback dirty changes then retry save`, and the `restore composite id
after rollback` / `restore id after rollback` pair) stays green.
- [ ] `pnpm parity:api:calls` adds zero rows for `transactions.ts`.
