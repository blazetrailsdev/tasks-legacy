---
title: "Retire the remaining _attributeDefinitions registry (user declarations live in the pending queue)"
status: ready
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`retire-attribute-definitions-registry-for-default-attributes` (#6948) stopped
`_attributeDefinitions` carrying schema-reflected columns: `applyColumnsHash`
now only stashes `columns_hash` minus `ignored_columns`
(`vendor/rails/activerecord/lib/active_record/model_schema.rb:592-594`) and
`_defaultAttributes` seeds from `columns_hash` alone
(`vendor/rails/activerecord/lib/active_record/attributes.rb:241-252`).

What remains is the other half: the registry still exists, holding **user
declarations only**. Rails has no such registry at all — a user declaration
lives solely in the pending-modification queue
(`vendor/rails/activemodel/lib/active_model/attribute_registration.rb:53-72`),
and everything else is derived from `_default_attributes` / `attribute_types`.

Writers still in place:

- `packages/activerecord/src/attributes.ts` — `defineAttribute`'s copy-on-write
  `_attributeDefinitions.set(...)`. Rails' `define_attribute` writes only
  `attribute_types[name] = cast_type` plus `define_default_attribute`
  (`attributes.rb:231-238`).
- `packages/activemodel/src/attributes.ts` — `attribute()`'s
  `_attributeDefinitions.set(...)` (:96-134), beside the queue push it already
  does. Rails' `attribute` pushes to the queue and nothing else
  (`attribute_registration.rb:12-20`).

Readers still in place:

- `packages/activerecord/src/model-schema.ts` — the synthesized `columnsHash`
  fallback for tableless models, the same synthesis in `loadSchemaBang`, the
  `createTable` DDL helper, and `reconcileVirtualAttributes` (see
  [[retire-virtual-attribute-reconciliation]], which removes that reader).
- `packages/activerecord/src/base.ts` — `ensureSchemaLoaded`'s
  declaration-scan bail.

## Converged shape

Delete `_attributeDefinitions` and the `AttributeDefinition` interface. Every
surviving reader resolves through `_default_attributes` / `attribute_types`,
which already carry the declarations via the replay. The tableless-model
`columnsHash` synthesis is the one that needs real thought: Rails has no such
fallback because `columns_hash` is a DB read, so it likely folds into
`attribute_names` rather than a synthesized column set.

This is the owner half of
`converge-attribute-definitions-activerecord-owners` (RFC 0078), whose
`blocked-by` predates #6948 and no longer describes the state of `main` —
re-triage that story against this one rather than duplicating it.

## Acceptance criteria

- [ ] `_attributeDefinitions` and `AttributeDefinition` are deleted from
      `packages/activemodel/src/attributes.ts`, `packages/activemodel/src/model.ts`
      and `packages/activerecord/src/attributes.ts`.
- [ ] No reader in `activerecord` or `activemodel` references the map.
- [ ] `pnpm parity:api:extra` shows no new novel surface; parity deltas
      non-negative; `pnpm parity:api:calls` / `:args` clean.
