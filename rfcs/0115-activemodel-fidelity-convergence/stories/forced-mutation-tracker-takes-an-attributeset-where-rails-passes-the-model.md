---
title: "forced-mutation-tracker-takes-an-attributeset-where-rails-passes-the-model"
status: claimed
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-24T22:54:10Z"
assignee: "forced-mutation-tracker-takes-an-attributeset-where-rails-passes-the-model"
blocked-by: null
closed-reason: null
---

## Context

`ForcedMutationTracker` is the tracker Rails builds when a `Dirty`-including
object has no `@attributes` — `ActiveModel::ForcedMutationTracker.new(self)`
(`vendor/rails/activemodel/lib/active_model/dirty.rb:385`). It is handed the
**model**, not an `AttributeSet`, which is why Rails overrides `fetch_value`:

```ruby
def fetch_value(attr_name)
  attributes.send(:_read_attribute, attr_name)
end
```

(`vendor/rails/activemodel/lib/active_model/attribute_mutation_tracker.rb:140-142`)

`packages/activemodel/src/attribute-mutation-tracker.ts:148` has no such
override, so `ForcedMutationTracker` inherits
`AttributeMutationTracker#fetchValue` (`:133`), which calls
`this.attributes.fetchValue(...)` — an `AttributeSet` method the model does not
answer. Its constructor is also typed `AttributeSet`, so
`packages/activemodel/src/dirty.ts`'s `mutations_from_database` reaches it
through `this as unknown as AttributeSet`.

The arm is unreachable today: trails' `Model` always initialises
`_attributes` (`model.ts`), so `mutations_from_database` always takes the
`AttributeMutationTracker` branch, and PR #7004 (which converged `Dirty` onto
the two trackers) documents that at the call site rather than fixing it. The
three `ForcedMutationTracker` tests in
`packages/activemodel/src/attribute-mutation-tracker.test.ts:161-193` construct
it with an `AttributeSet`, which is the shape Rails never passes — they move
with the fix.

## Acceptance criteria

- `ForcedMutationTracker` takes the host Rails passes it (a `_read_attribute`
  answerer), not an `AttributeSet`, and overrides `fetchValue` as
  `attribute_mutation_tracker.rb:140-142` does.
- `dirty.ts`'s `mutations_from_database` builds it without a cast.
- The three `ForcedMutationTracker` tests construct it with a model-shaped
  host; test names unchanged.
- `pnpm parity:api:calls`, `pnpm parity:api:calls:args`,
  `pnpm parity:api:extra --package activemodel` non-regressing.
