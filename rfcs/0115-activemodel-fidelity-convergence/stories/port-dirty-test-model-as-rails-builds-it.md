---
title: "Port dirty_test.rb's DirtyModel as Rails builds it, so the file exercises ForcedMutationTracker"
status: draft
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `DirtyTest::DirtyModel`
(`vendor/rails/activemodel/test/cases/dirty_test.rb:6-43`) includes
`ActiveModel::API` + `ActiveModel::Dirty` and **nothing else** — no
`ActiveModel::Attributes`. It seeds plain ivars in `initialize`, exposes them
with `attr_reader`, and every writer calls `*_will_change!` by hand:

```ruby
def initialize
  @name = nil; @color = nil; @size = nil; @status = "initialized"
end
attr_reader :name, :color, :size, :status
def name=(val)
  name_will_change!
  @name = val
end
```

Because that model has no `@attributes`, `Dirty#mutations_from_database`
(`vendor/rails/activemodel/lib/active_model/dirty.rb:382-388`) takes its
**second** arm and builds an `ActiveModel::ForcedMutationTracker`
(`:385`). The whole of `dirty_test.rb` therefore exercises
`ForcedMutationTracker` — `force_change`, `finalize_changes`, the
`forced_changes`-keyed `attr_names`, and `changed_in_place?` returning a flat
`false` (`attribute_mutation_tracker.rb:91-154`).

`packages/activemodel/src/dirty.test.ts` ports those test names onto a
`Model` subclass with `this.attribute(...)` declarations, seeded through the
constructor. That model always has `_attributes`, so it takes the **first**
arm and builds an `AttributeMutationTracker`. The result is that trails'
`dirty_test.rb` port never runs a single line of `ForcedMutationTracker`,
while `attributes_dirty_test.rb`
(`vendor/rails/activemodel/test/cases/attributes_dirty_test.rb:6-21`) — whose
`DirtyModel` _does_ include `ActiveModel::Attributes` — is ported onto the
same shape, so the two files test the same path as each other instead of the
two different ones Rails covers.

PR #7004 (which converged `Dirty` onto the two mutation trackers) added a
`changes_applied` call to each fixture so the constructor seeding reads as the
baseline; that keeps the assertions honest but does not close this gap.

## Acceptance criteria

- `dirty.test.ts`'s model mirrors `dirty_test.rb:6-43`: `ActiveModel::API` +
  `Dirty` only, ivar-backed readers, hand-written writers calling
  `*_will_change!`, and a `save` that calls `changes_applied`. It must NOT
  declare attributes through `ActiveModel::Attributes`.
- `mutations_from_database` on it resolves to `ForcedMutationTracker`, so the
  file exercises that class the way Rails does.
- The per-fixture `changes_applied` calls PR #7004 added to `dirty.test.ts`
  disappear with the constructor seeding they compensate for.
- `attributes-dirty.test.ts` keeps the `Attributes`-including shape
  (`attributes_dirty_test.rb:6-21`) so the two files stay distinct.
- Test names unchanged.
- Depends on
  `0115-activemodel-fidelity-convergence/forced-mutation-tracker-takes-an-attributeset-where-rails-passes-the-model`,
  which fixes `ForcedMutationTracker`'s host type and `fetch_value`
  (`attribute_mutation_tracker.rb:140-142`) — without it the second arm is not
  actually usable.
