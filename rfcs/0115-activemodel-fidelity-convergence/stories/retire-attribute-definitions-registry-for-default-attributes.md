---
title: "Retire _attributeDefinitions in favour of _default_attributes + the pending queue"
status: in-progress
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: api-compare
packages: ["activemodel", "activerecord"]
deps: []
deps-rfc: []
est-loc: 500
pr: 6948
claim: "2026-08-23T20:34:25Z"
assignee: "retire-attribute-definitions-registry-for-default-attributes"
blocked-by: null
closed-reason: null
---

## Context

Rails keeps no attribute-definition registry. A class-level attribute lives in
exactly two places: `_default_attributes` (an `ActiveModel::AttributeSet` built
by seeding `columns_hash` and replaying the pending-modification queue —
`activerecord/lib/active_record/attributes.rb:241-252`) and the queue itself
(`activemodel/lib/active_model/attribute_registration.rb:53-72`). Provenance is
carried by _which `Attribute` subclass_ the seed/replay produces, never by a
stored field.

trails instead keeps `_attributeDefinitions`, a `Map<string,
AttributeDefinition>` that BOTH `attribute()` and schema reflection
(`applyColumnsHash` in `packages/activerecord/src/model-schema.ts`) write into.
Because reflection re-registers over user declarations, several call sites need
to know which rows are user-declared.

PR for `retire-attribute-definition-provenance-flag` removed the stored
`userProvidedDefault` field: those call sites now read the provenance back off
the pending-modification queue via `pendingAttributeDeclarationQ` /
`pendingAttributeTypeQ` (`packages/activemodel/src/attribute-registration.ts`),
which is where Rails reads it. Those two helpers are the remaining
`@noRailsEquivalent` surface in that file, and they exist ONLY because the
registry exists.

Readers to retire along with the registry:

- `packages/activerecord/src/model-schema.ts` — `scrubSchemaSourcedDefinitions`,
  `applyColumnsHash` (ignored-columns delete + the preserve-user-override arm),
  `reconcileVirtualAttributes`
- `packages/activerecord/src/attributes.ts` — `_defaultAttributes` phase 1,
  `defineAttribute`
- `packages/activerecord/src/base.ts` — the virtual-reconcile branch in
  `ensureSchemaLoaded`

## Acceptance criteria

- [ ] `_attributeDefinitions` no longer carries schema-reflected columns; the
      column seed comes from `columns_hash` at `_default_attributes` time as
      `attributes.rb:241-252` does.
- [ ] `pendingAttributeDeclarationQ` and `pendingAttributeTypeQ` are deleted
      from `packages/activemodel/src/attribute-registration.ts` and its
      `index.ts` re-export.
- [ ] `pnpm parity:api:extra --package activemodel` shows
      `attribute-registration.ts` with no `@noRailsEquivalent` tag for either.
- [ ] Parity deltas non-negative; `pnpm parity:api:calls` / `:args` clean.
