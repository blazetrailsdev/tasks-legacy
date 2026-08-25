---
title: "AssociationDefinition.type holds the macro where Rails' reflection.type is the polymorphic column"
status: draft
updated: 2026-08-12
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

Surfaced in PR #6409 while converging `Association#initialize_attributes` to
Rails' direct `[reflection.foreign_key, reflection.type].compact`
(`vendor/rails/activerecord/lib/active_record/associations/association.rb:219`).

`AssociationDefinition` (`packages/activerecord/src/associations.ts:163-188`)
declares

```ts
type: "belongsTo" | "hasOne" | "hasMany" | "hasAndBelongsToMany";
```

i.e. the MACRO name — while every Rails reflection spells that `macro`
(`reflection.rb`, `MacroReflection#macro`) and reserves `type` for the
polymorphic `*_type` COLUMN (`AssociationReflection#type` →
`foreign_type`, ours at `packages/activerecord/src/reflection.ts:1110`). The
interface's own doc comment already records the collision and tells bodies
branching on the macro to read the optional `macro?` field instead.

The consequence is that one field name means two different things depending on
which object a body happens to hold: on the rich reflection
`reflection.type` is `"taggable_type"`, on a lightweight definition literal it
is `"hasMany"`. PR #6409's `initializeAttributes` read is correct on every live
path (all callers resolve through `record.association(name)`, which builds
against the rich reflection), but the type system actively misdescribes it —
`this.reflection.type` is statically the macro union there.

The lightweight literals are built at
`associations/has-many-through-association.ts:1198`,
`associations/has-one-through-association.ts:789`,
`associations/collection-proxy.ts:915`,
`associations/has-many-association.ts:621` and
`test-helpers/find-collection-target.ts:25`.

## Converged shape

`AssociationDefinition.type` means what `AssociationReflection#type` means —
the polymorphic foreign-type column (`string | null`) — and the macro lives
only on `macro`, as in Rails. The five ad-hoc literals above pass
`macro: "hasMany"` / `"hasOne"` rather than `type:`; every reader that branches
on the macro is already migrated to `macro`.

## Acceptance criteria

1. `AssociationDefinition.type` is typed and documented as the polymorphic
   foreign-type column, citing `reflection.rb` / `association.rb:219`.
2. No production body reads `.type` expecting a macro name; the five
   `_buildAssociationInstance` literals pass `macro`.
3. `parity:api:extra --package activerecord` gains no row;
   `parity:api:calls` / `:args` non-regressive.
4. Association, through-association, polymorphic, STI and nested-attributes
   suites green on all three adapter lanes.
