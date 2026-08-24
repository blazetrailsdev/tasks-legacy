---
title: "changed_in_place? for an immutable JS scalar is reachable only by writing Attribute's private fields"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

`Attribute#changed_in_place?`
(`vendor/rails/activemodel/lib/active_model/attribute.rb:70`) is
`has_been_read? && type.changed_in_place?(original_value_for_database, value)`.
Ruby reaches that state by mutating the cast value the `Attribute` already
holds, leaving the same `Attribute` — and its
`original_value_for_database` — in the set:

```ruby
@aircraft.name.downcase!     # normalized_attribute_test.rb:36
```

JS strings, numbers and Dates are immutable, so trails has **no public route**
to that state for a scalar attribute. Writing through the set instead produces
an `Attribute::WithCastValue`, whose `changed_in_place?` is `false` by
definition (`attribute.rb:243-245`).

Before PR #7004 this was papered over: the invented `DirtyTracker.changedInPlace`
carried a `@missingRailsCall changed_in_place?` fallback that compared values
directly. #7004 deleted that tracker, so `changed_in_place?` is now the plain
`attributes[attr_name].changed_in_place?`
(`attribute_mutation_tracker.rb:50-52`) — correct, but it left
`packages/activerecord/src/normalized-attribute.test.ts` unable to set up its
fixture. That test now reaches into `Attribute`'s private fields:

```ts
const nameAttr = aircraft._attributes.getAttribute("name") as unknown as {
  _value: unknown;
  _hasValue: boolean;
};
nameAttr._value = "fly high";
nameAttr._hasValue = true;
```

That is the only setup in the tree that pokes `Attribute` internals, and it
means the production path it covers —
`Normalization#normalize_changed_in_place_attributes`
(`vendor/rails/activerecord/lib/active_record/normalization.rb:112-116`),
which runs on `before_validation` for every normalized attribute — is
reachable in tests only by surgery. Any refactor of `Attribute`'s value
caching silently breaks the fixture rather than the feature.

## Acceptance criteria

- The fixture reaches the changed-in-place state through a supported route,
  not by assigning `_value` / `_hasValue`. Options to weigh, in fidelity
  order: a mutable-typed attribute (serialized Hash/Array), where JS genuinely
  mutates in place exactly as Ruby does and no reach-in is needed; or an
  `Attribute` affordance that mirrors what Ruby's in-place mutation leaves
  behind, if the string case must stay covered.
- Whatever route is chosen, `changed_in_place?` still resolves through
  `attributes[attr_name].changed_in_place?` — do NOT reintroduce a
  value-comparison fallback in the tracker; that is the deviation #7004
  removed.
- `normalized-attribute.test.ts`'s test names stay verbatim
  (`vendor/rails/activerecord/test/cases/normalized_attribute_test.rb:35-41`).
- If the string case turns out to be genuinely unreachable in TS, say so at
  the call site with a cited `@noRailsEquivalent` / `@missingRailsCall` on the
  fixture rather than leaving a bare private-field write.
