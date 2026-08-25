---
title: "Converge DirtyTracker#changedInPlace onto Attribute#changed_in_place?"
status: closed
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
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
closed-reason: "Premise gone on origin/main. The invented DirtyTracker was retired (PR #7004 converged Dirty onto the two Rails mutation trackers): git grep 'DirtyTracker' origin/main -- packages/activemodel/src returns zero hits. AttributeMutationTracker#changedInPlace is now Rails' one line verbatim -- 'return this.attributes.getAttribute(attrName).changedInPlace()' (attribute-mutation-tracker.ts:100-101 = attribute_mutation_tracker.rb:50-52); Forced/Null trackers return false as Rails does (:170, :255). Dirty#attributeChangedInPlace just delegates (dirty.ts:164-169). No '@missingRailsCall changed_in_place?' tag survives anywhere in packages/."
---

## Context

`DirtyTracker#changedInPlace` (`packages/activemodel/src/dirty.ts`) carries a
`@missingRailsCall changed_in_place? — PERMANENT` and a two-branch body where
Rails has one line.

Rails (`activemodel/lib/active_model/attribute_mutation_tracker.rb:50-52`):

```ruby
def changed_in_place?(attr_name)
  attributes[attr_name].changed_in_place?
end
```

and `Attribute#changed_in_place?` (`activemodel/lib/active_model/attribute.rb:70-72`):

```ruby
def changed_in_place?
  has_been_read? && type.changed_in_place?(original_value_for_database, value)
end
```

trails asks the Attribute only when `type.isMutable()`, and otherwise falls
back to comparing the value against `changes[name][1]` / `attributeWas(name)`.

The mutable half is already Rails' body and is pinned by
`packages/activerecord/src/dirty.trails.test.ts`
("attributeChangedInPlace is true for a serialized attribute mutated in place").
This story is about deleting the immutable half.

### What is known

Landed in #6990. Making the guard unconditional — i.e. `return
this._attrs?.getAttribute(name).changedInPlace() ?? false`, Rails' whole body —
reds two cases in `packages/activerecord/src/normalized-attribute.test.ts`:

- `normalizes changed-in-place value before validation` (`:79`)
- `minimizes number of times normalization is applied` (`:154`)

Both mirror `activerecord/test/cases/normalized_attribute_test.rb:35-41` and
`:99-111`, and both are green on the shipped two-branch body.

The first is understood and is a genuine language shortcoming: Rails reaches an
in-place change by mutating the cast value the Attribute already holds
(`@aircraft.name.downcase!`, `normalized_attribute_test.rb:36`). JS strings are
immutable, so the port's stand-in is `_attributes.set(name, ...)`, which yields
an `Attribute::WithCastValue` whose `changed_in_place?` is `false` by
definition (`attribute.rb:243-245`). That arm may be irreducible.

The second is NOT understood and is the reason this story exists rather than a
permanent ratification. `has_been_read?` short-circuits Rails'
`changed_in_place?`, and a bare write does NOT mark an attribute read in trails
either — verified by probe: `new NormalizedAircraft(); a.name = "fly HIGH"` then
`_attributes.getAttribute("name").hasBeenRead()` is `false`, ctor `FromUser`.
So the obvious explanation (trails marks reads at write time, Rails does not)
is ruled out. Root-cause it before concluding anything.

## Acceptance criteria

- Root-cause the `minimizes number of times normalization is applied` failure
  under an unconditional `Attribute#changedInPlace()` guard. It is a divergence
  in `attribute.rb`'s port (`originalValueForDatabase`, `hasBeenRead`, or the
  normalized-type `isChangedInPlace`), not in `dirty.rb`'s — fix it there.
- `DirtyTracker#changedInPlace` becomes
  `attributes[attr_name].changed_in_place?` and nothing else, or the story is
  `pnpm tasks block`ed with the specific irreducible arm named and the
  `@missingRailsCall` reason narrowed to exactly that arm.
- The `@missingRailsCall changed_in_place?` tag is deleted if the body
  converges.
- `normalized-attribute.test.ts` and `dirty.trails.test.ts` stay green; the
  serialized in-place case keeps its coverage.
