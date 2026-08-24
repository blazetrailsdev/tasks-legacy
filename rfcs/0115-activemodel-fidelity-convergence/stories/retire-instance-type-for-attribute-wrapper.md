---
title: "Model#typeForAttribute is invented instance surface"
status: in-progress
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7006
claim: "2026-08-24T20:39:29Z"
assignee: "converge-attribute-method-predicate-to-rails-body"
blocked-by: null
closed-reason: null
---

## Context

`Model` carries an INSTANCE `typeForAttribute`
(`packages/activemodel/src/model.ts:797-804`):

```ts
/**
 * Returns the type of the attribute (the Type object).
 *
 * Mirrors: ActiveModel::Attributes#attribute_for_inspect
 */
typeForAttribute(name: string, block?: () => Type): Type {
  return (this.constructor as typeof Model).typeForAttribute(name, block);
}
```

Rails has no such instance method. `type_for_attribute` is defined ONLY inside
`ActiveModel::AttributeRegistration::ClassMethods`
(`vendor/rails/activemodel/lib/active_model/attribute_registration.rb:43`); a
`grep -rn "def type_for_attribute"` over `activemodel/lib` and
`activerecord/lib` returns that one plus three unrelated
`Arel::Table` / `TypeCaster` definitions, none of them on a model instance.

Two separate problems:

1. **Invented instance surface.** The method is a delegation wrapper to the
   class method that Rails does not have, so a caller can write
   `record.typeForAttribute(...)` where Rails requires
   `record.class.type_for_attribute(...)`.
2. **The JSDoc cites the wrong Rails method.** `attribute_for_inspect`
   (`activerecord/lib/active_record/attribute_methods.rb`) is an unrelated
   method that returns a truncated STRING for inspection, not a `Type`. The
   `Mirrors:` line is wrong and would credit the wrong Rails method to any
   reader or tool that trusts it.

PR #7000 deleted the shadowed class-side `typeForAttribute` from `model.ts`
(it now arrives via `extend()` from `attribute-registration.ts`) but left this
instance wrapper in place — it was out of that story's scope, which was the
class half.

## Converged shape

Delete the instance method. Callers go through the class method, as Rails does.
`pnpm parity:api:extra --package activemodel` should lose the row.

If a caller genuinely needs it (check `packages/activerecord` — `type_for_attribute`
is reached from schema/type code), route that caller through
`(this.constructor as typeof Model).typeForAttribute(...)` at the call site
rather than reinstating the wrapper.

## Acceptance criteria

- `model.ts` defines no instance `typeForAttribute`; every caller reaches the
  class method.
- No `Mirrors:` line anywhere in the tree cites `attribute_for_inspect` for this
  method.
- `pnpm parity:api:extra --package activemodel` loses the row; parity deltas
  non-negative.
