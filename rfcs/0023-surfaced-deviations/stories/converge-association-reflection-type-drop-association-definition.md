---
title: "Type Association#reflection as the reflection and retire the AssociationDefinition stand-in"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced by PR #6367, which made `Association#reflection` hold the rich
reflection at runtime (`associations.rb:290-296` —
`reflection.association_class.new(self, reflection)`).

The declared TYPE is still `AssociationDefinition`
(`packages/activerecord/src/associations.ts:163`), a trails invention with no
Rails counterpart:

```ts
export interface AssociationDefinition {
  type: "belongsTo" | "hasOne" | "hasMany" | "hasAndBelongsToMany";
  name: string;
  options: AssociationOptions & { joinTable?: string };
}
```

Rails has only `AssociationReflection` / `ThroughReflection`. Because the
declared type lags the runtime value, #6367 had to bolt the Rails-named
reflection members onto the interface as optionals (`macro`, `foreignKey`,
`associationPrimaryKey`, `counterCacheColumn`, `hasCachedCounter`,
`isCounterMustBeUpdatedByHasMany`, `inverseName`) purely so the bodies could
read them, and `AssociationScope.scope(this as unknown as AssociationScopeable)`
still needs a cast (`associations/association.ts:379`).

The blocker is `options`: `AssociationDefinition.options` is the typed
`AssociationOptions`, while `MacroReflection#options` is
`Record<string, unknown>` (`reflection.ts:62`, `:575`), so retyping the field
turns ~70 `this.reflection.options.X` reads into `unknown`.

Two further reads inherited the same shape and are worth fixing in the same
pass, since both re-derive what the reflection already answers:

- `BelongsToAssociation#foreignKeyName` / `#foreignKeyNames`
  (`belongs-to-association.ts:305-315`) exist only to array-wrap
  `reflection.foreign_key`; Rails has no such helpers.
- `CollectionAssociation#associationPrimaryKey`
  (`collection-association.ts:249`) appends `?? klass.primaryKey ?? "id"`,
  where Rails' `collection_association.rb#ids_reader` just calls
  `reflection.association_primary_key`.

## Converged shape

`Association#reflection` typed as `AssociationReflection | ThroughReflection`
(type-only import, so no new module-eval edge), `AssociationDefinition` deleted
or reduced to the declaration-macro input it actually is, the optional
reflection members removed from it, and the `AssociationScopeable` cast at
`association.ts:379` dropped.

## Acceptance criteria

- [ ] `Association#reflection` is declared as the reflection type; no optional
      Rails-named members remain on `AssociationDefinition`.
- [ ] The `as unknown as AssociationScopeable` cast at `association.ts:379` is
      gone.
- [ ] `foreignKeyName` / `foreignKeyNames` and the `associationPrimaryKey`
      fallback are removed in favour of the reflection's own accessors.
- [ ] `pnpm parity:api:extra --package activerecord` does not grow; any row the
      convergence retires is deleted by hand (only-shrink, no `--write`).
- [ ] AR suites pass on all three adapter lanes.
