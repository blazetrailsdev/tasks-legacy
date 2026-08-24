---
title: "delete-attribute-definitions-map-and-schema-sourced-scrub"
status: closed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Both halves resolved elsewhere. AC1 already landed: 'scrubSchemaSourcedDefinitions' returns ZERO hits on origin/main (git grep -n scrubSchemaSourcedDefinitions origin/main -- packages), and model-schema.ts's reloadSchemaFromCache now only nils memos as Rails' model_schema.rb:553-568 does. AC2 (delete _attributeDefinitions from SchemaHost and every remaining reader) is verbatim the scope of 0115/retire-remaining-attribute-definitions-registry, whose body enumerates the same surviving readers (model-schema.ts synthesized columnsHash fallback, loadSchemaBang, createTable, reconcileVirtualAttributes, base.ts ensureSchemaLoaded). Closing to avoid two stories owning the same deletion; the remaining work is tracked in 0115."
---

## Context

Remainder of `delete-schema-revision-and-decorator-replay-machinery` (PR that
deleted the `_schemaRevision` epoch, `schemaEpoch`, `schemaStaleAgainstAncestors`,
`ownProp` and the `blazetrails/schema-memo-read-through-guard` ESLint rule).

Its two other acceptance criteria could not ship: they are premised on
`_attributeDefinitions` being gone, and the `converge-attribute-definitions-*`
reader slices are still in flight, so the map is still read by
`packages/activemodel/src/{attributes,model,serialization,attribute-registration}.ts`
and `packages/activerecord/src/{attributes,inheritance,attribute-methods}.ts`.

Left standing in `packages/activerecord/src/model-schema.ts`:

- `scrubSchemaSourcedDefinitions` — hand-partitions `_attributeDefinitions` by
  provenance (`pendingAttributeDeclarationQ`) on every
  `reloadSchemaFromCache`. Rails' per-class replay of
  `pending_attribute_modifications` does this for free
  (`vendor/rails/activemodel/lib/active_model/attribute_registration.rb:80-99`);
  `reload_schema_from_cache` (`activerecord/lib/active_record/model_schema.rb:553-568`)
  only nils ivars. Deleting it on `origin/main` reds 14 tests across
  `persistence.test.ts`, `locking.test.ts` and `migration.test.ts` — measured,
  not predicted — because the reflected columns still live in the map.
- `_attributeDefinitions` itself.

## Acceptance criteria

- [ ] `scrubSchemaSourcedDefinitions` is deleted from `model-schema.ts`, with
      `reloadSchemaFromCache` left nilling memos only, as Rails does.
- [ ] `_attributeDefinitions` is deleted from `SchemaHost` and from every
      remaining reader.
- [ ] No regression in `persistence`, `locking`, `migration`, `inheritance`,
      `attributes`, `model-schema`, `enum`, `dirty`, `sti/` and `encryption/`.

## Dependencies

After the `converge-attribute-definitions-*` reader slices.
