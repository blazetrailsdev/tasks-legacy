---
title: "Converge the HABTM macro onto Rails' builder sequence"
status: done
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: 6900
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the `associations.rb` macro rows in PR #6727
(`wave-4d-associations-residue`). Three of the four macros now end the way
Rails does — build the reflection, then register it:

```ruby
def has_many(name, scope = nil, **options, &extension)
  reflection = Builder::HasMany.build(self, name, scope, options, &extension)
  Reflection.add_reflection self, name, reflection
end
```

(`vendor/rails/activerecord/lib/active_record/associations.rb:1302-1305`;
`has_one` :1498-1500, `belongs_to` :1689-1691.)

`has_and_belongs_to_many` could not follow, so its `add_reflection` call-set row
is the one macro row left in `call-mismatches-exclude/activerecord/associations.json`.

Rails' HABTM macro (`associations.rb:1870-1906`) is a readable sequence of
builder calls:

```ruby
habtm_reflection = ActiveRecord::Reflection::HasAndBelongsToManyReflection.new(name, scope, options, self)
builder = Builder::HasAndBelongsToMany.new name, self, options
join_model = builder.through_model
const_set join_model.name, join_model
private_constant join_model.name
middle_reflection = builder.middle_reflection join_model
Builder::HasMany.define_callbacks self, middle_reflection
Reflection.add_reflection self, middle_reflection.name, middle_reflection
middle_reflection.parent_reflection = habtm_reflection
# ... destroy_associations module, hm_options allowlist ...
has_many name, scope, **hm_options, &extension
_reflections[name].parent_reflection = habtm_reflection
```

trails collapses all of it into `HasAndBelongsToMany._build`
(`packages/activerecord/src/associations/builder/has-and-belongs-to-many.ts`),
a single ~200-line private method that returns `void`. It registers BOTH the
middle and the public reflection itself via `Reflection.addReflection`, and it
never calls the `has_many` macro — it hand-builds the through reflection with a
`HABTM_FORWARDED_KEYS` allowlist standing in for Rails' `hm_options`. Because
`_build` returns nothing, `Associations.hasAndBelongsToMany` has no reflection
to hand to `Reflection.addReflection`, which is exactly why that row survives.

The class's public shape is already Rails-named — `throughModel()`,
`middleReflection(joinModel)`, and (as of #6727) the `middleOptions` /
`belongsToOptions` privates all exist and match
`builder/has_and_belongs_to_many.rb:59-102`. They are simply not what `_build`
calls; `middleReflection` currently has no caller at all in `src/`.

## Converged shape

Delete `_build` and its `deps` parameter object. `Associations.hasAndBelongsToMany`
(`associations.ts`) becomes the Rails macro body, calling the already-correct
builder methods in Rails' order and finishing with the real `hasMany` macro, so
`add_reflection` is reached the same way it is for the other three macros. The
existing trails-only pieces that have no Rails counterpart — the
`destroyAssociations` wrapper-layering guard and the `anonymousClass` join-model
handle (tracked by `converge-constantize-ignores-private-constants`) — stay, but
move to the macro alongside the Rails lines they stand in for.

Watch out for: `throughModel()` currently returns a plain object literal, not a
`Base` subclass, while `_build` uses the real `createHabtmJoinModel` — those two
join-model constructions have to be reconciled into one before the sequence can
be rewired.

## Acceptance criteria

- [ ] `Associations.hasAndBelongsToMany` mirrors `associations.rb:1870-1906`:
      `through_model` → `middle_reflection` → `define_callbacks` →
      `add_reflection` → `has_many`, with the Rails locals (`habtmReflection`,
      `builder`, `joinModel`, `middleReflection`, `hmOptions`).
- [ ] `HasAndBelongsToMany._build` and its `deps` injection object are gone;
      `throughModel()` and `middleReflection()` are the live code paths.
- [ ] `hmOptions` is Rails' key list (`associations.rb:1899`), with any trails
      addition justified at the call site with a Rails cite.
- [ ] The `has_and_belongs_to_many -> add_reflection` row is deleted from
      `call-mismatches-exclude/activerecord/associations.json` by hand via
      `serializeBaseline`, then `pnpm parity:api:calls:tighten activerecord/associations.json`.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no new row.
- [ ] `has-and-belongs-to-many-associations.test.ts` and the HABTM extension /
      fixture suites green on SQLite, PostgreSQL and MySQL/MariaDB.
