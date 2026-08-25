---
title: "select the autosave callback branch with reflection predicates, not duck-typing ladders"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converting `addAutosaveAssociationCallbacks` to the `this`-typed
idiom in #6469. The body's two macro branches are selected by duck-typing
ladders that Rails does not have:

`packages/activerecord/src/autosave-association.ts`:

```ts
const isCollection: boolean =
  typeof reflection.isCollection === "function"
    ? reflection.isCollection()
    : reflection.collection === true ||
      reflection.macro === "hasMany" ||
      reflection.macro === "hasAndBelongsToMany" ||
      reflection.type === "hasMany" ||
      reflection.type === "hasAndBelongsToMany";
const isHasOne: boolean =
  typeof reflection.hasOne === "function"
    ? reflection.hasOne()
    : reflection.hasOne === true || reflection.macro === "hasOne" || reflection.type === "hasOne";
```

Rails (`activerecord/lib/active_record/autosave_association.rb:190-214`) simply
asks the reflection:

```ruby
if reflection.collection?
  ...
elsif reflection.has_one?
```

`ActiveRecord::Reflection::AssociationReflection` defines `collection?` and
`has_one?` as plain predicates, so the branch is one call with no fallbacks.
The ladders here are a trails invention that silently accepts four different
shapes per branch, which means a reflection object missing the predicate still
routes somewhere rather than failing loudly.

The same ladder is duplicated in `defineAutosaveValidationCallbacks` in the
same file (`autosave_association.rb:219-233` is likewise a bare
`reflection.collection?` / `reflection.has_one?`).

## Converged shape

Both bodies call `reflection.isCollection()` and `reflection.hasOne()`
directly, per `docs/ruby-ts-conventions.md`'s predicate mapping. If some
reflection-like object reaching these call sites genuinely lacks the
predicates, that object is the bug — fix it at its source rather than widening
the branch here.

## Acceptance criteria

- [ ] `addAutosaveAssociationCallbacks` and `defineAutosaveValidationCallbacks`
      each select their branch with a single predicate call, matching
      `autosave_association.rb:190` and `:222`.
- [ ] The `macro` / `type` / boolean-field fallbacks are gone from both.
- [ ] `autosave-association.test.ts` and the has-one / has-many / belongs-to
      association suites pass on all three adapters.
