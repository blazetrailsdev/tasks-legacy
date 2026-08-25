---
title: "Drop the last trails-only input to the new() STI gate (the stiEnabled disjunct), leaving _has_attribute? alone"
status: blocked
updated: 2026-08-25
rfc: "0123-blocked-convergence-holding"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: "2026-08-18T20:31:56Z"
assignee: "wave-4c-ar-core-residue-attributes"
blocked-by: 'Blocked on a sync schema signal that does not exist. Measured on origin/main + this branch: inheritance-sti-new-gate.trails.test.ts''s precondition still holds — classHasAttribute(VerySpecialClient, "type") === false at new() with no connection established, so the eager-warm work (#3373) has NOT closed the cold window. classHasAttribute (inheritance.ts:430) answers from _attributeDefinitions or columnNames(), and columnNames() cannot reflect synchronously; Rails'' _has_attribute? is never cold because attribute_types loads the schema synchronously (inheritance.rb:61 -> model_schema.rb load_schema). Dropping the !stiEnabled disjunct therefore makes an STI leaf with an unreflected type column build as-is where Rails raises SubclassNotFound. Converging needs the schema cache readable synchronously at construction time (RFC 0023 async/sync), or the generation-time signal Company.inheritanceColumn = "type" (test-helpers/models/company.ts:644 — itself a trails-only line Rails'' company.rb does not have) replaced by real reflection. Note the sibling deviation in the same closure, the fromScope/namesSelfOrStiAncestor escape, DID converge in PR for converge-new-sti-scope-source-ancestor-raise.'
closed-reason: null
---

## Context

Residual left by `converge-new-sti-gate-onto-has-attribute-alone` (PR #6713),
which converged the rest of the `new()` STI dispatch gate.

Rails gates the ENTIRE `new` STI dispatch on one signal —
`_has_attribute?(inheritance_column)`
(`vendor/rails/activerecord/lib/active_record/inheritance.rb:61`, inside
`ClassMethods#new` at `:56-78`). trails'
`subclassFromAttributesForNew` (`packages/activerecord/src/inheritance.ts:1149`)
still reads a two-term disjunct:

```ts
if (!classHasAttribute(modelClass, col) && !stiEnabled(modelClass)) return null;
```

The `!stiEnabled(modelClass)` arm is the last trails-only input. It exists
because Rails' `attribute_types` loads the schema _synchronously_ on first
touch (`inheritance.rb:61` -> `model_schema.rb`'s `load_schema`), so
`_has_attribute?` is never cold; a TypeScript constructor cannot query the
database synchronously, so `classHasAttribute`
(`packages/activerecord/src/inheritance.ts:430`) can still be `false` at `new`
for a model with a real `type` column whose schema has not reflected yet.
Without the arm, an STI _leaf_ whose `type` column had not reflected builds
as-is where Rails raises `SubclassNotFound` — the case pinned by
`packages/activerecord/src/inheritance-sti-new-gate.trails.test.ts`.

This is currently justified at the call site as a language shortcoming. That is
debt, not permission (CLAUDE.md).

## Converged shape

The gate reads `classHasAttribute(modelClass, inheritanceColumn)` alone, as
Rails reads `_has_attribute?` alone. That requires `classHasAttribute` to be
reliable at construction time for a model with a real `type` column.

Establish first exactly which construction paths still see a cold
`classHasAttribute` — `production-eager-schema-cache-warm-at-connection`
(#3373) already closes most of this window, so the residual may be narrow
enough that the arm is simply removable. Verify against the cold-leaf
precondition assertion in `inheritance-sti-new-gate.trails.test.ts`, which
asserts `classHasAttribute(VerySpecialClient, "type") === false` and would flip
to `true` if the window closes.

## Acceptance criteria

- [ ] The `new()` gate consults only the column-aware signal Rails consults; the
      `stiEnabled` disjunct is gone.
- [ ] `inheritance-sti-new-gate.trails.test.ts` still pins the cold-leaf raise
      (its precondition assertion updated to whatever is then true, not deleted).
- [ ] The call-site JSDoc/comment deviation note in
      `subclassFromAttributesForNew` is deleted, not reworded.
- [ ] Existing STI-at-new and STI-at-instantiate suites stay green;
      `parity:api` / `parity:test` deltas non-negative.

## Evidence from `vegetables-model-assigns-inheritance-column-rails-overrides-method` (PR pending, 2026-08-18)

`vegetables.ts` was converged onto Rails' `def self.inheritance_column` spelling
(vegetables.rb:6) — a `static override get inheritanceColumn()` — which leaves
`_inheritanceColumn` unset, so `stiEnabled(Vegetable)` is now **false** for that
tree while `Vegetable.inheritanceColumn` still answers `"custom_type"`.

STI dispatch was UNCHANGED: `inheritance.test.ts` stays green, including
`Vegetable.find(1) instanceof Cucumber` / `Vegetable.find(2) instanceof Cabbage`
(inheritance.test.ts:208-211) and the `becomes`/`becomesBang` cases. So the
database-row dispatch paths this tree exercises do NOT in fact depend on the
`stiEnabled` sentinel, which is direct evidence that the disjunct is reading a
trails-invented signal rather than a load-bearing one.
