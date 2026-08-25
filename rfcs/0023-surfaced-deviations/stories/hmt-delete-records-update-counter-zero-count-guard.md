---
title: "HMT delete_records guards update_counter behind count > 0; Rails does not (has_many_through_association.rb:168-173)"
status: draft
updated: 2026-08-19
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# HMT delete_records guards update_counter behind `count > 0`; Rails does not

## Context

`HasManyThroughAssociation#delete_records`
(`vendor/rails/activerecord/lib/active_record/associations/has_many_through_association.rb:168-173`)
ends with an UNGUARDED counter update:

```ruby
if through_reflection.collection? && update_through_counter?(method)
  update_counter(-count, through_reflection)
else
  update_counter(-count)
end
```

trails (`packages/activerecord/src/associations/has-many-through-association.ts`,
in `deleteRecords`) wraps the whole branch in `if (count > 0) { ... }`. So when
`count == 0` — a `delete_all` / `destroy` that matched no join rows — Rails
still calls `update_counter(0)`, which reaches `owner.increment!(counter, 0)`
and emits an `UPDATE`, while trails emits nothing.

The guard predates PR #6732, which converged the surrounding row
(`delete_records -> update_counter`) by routing both branches through the
inherited `HasManyAssociation#update_counter` and deleting the duplicate
`updateThroughCounterCache` / `throughCounterReflection` helpers. The guard was
deliberately left in place there to keep that PR's blast radius to the call-set
row; it is the remaining divergence in this body.

Converged shape: delete the `if (count > 0)` wrapper so both `update_counter`
calls run unconditionally, exactly as Rails' body does.

Note the interaction with `increment!`: since PR #6732 the wired instance
`increment!` is `Locking::Optimistic#increment!` (`optimistic.rb:63-70`), whose
locking arm bumps `lock_version` whenever `locking_enabled?`. A zero-delta
`update_counter` on a locked owner therefore also advances `lock_version` in
Rails. Verify that against MRI (`ruby` is on PATH) and against
`locking_test.rb` / `has_many_through_associations_test.rb` before assuming the
guard is pure noise — if removing it turns out to be observable, the fix is
still to match Rails, and the test that moves should be checked against its
Rails counterpart rather than adjusted.

## Acceptance criteria

- [ ] `if (count > 0)` removed; both `update_counter` branches run
      unconditionally, matching `has_many_through_association.rb:168-173`.
- [ ] Behaviour verified against the vendored Rails tests for
      `has_many :through` deletion with a counter cache, and against the
      optimistic-locking interaction described above.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
