---
title: "converge-attribute-definitions-activerecord-owners"
status: closed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps:
  - converge-attribute-definitions-activemodel-readers
  - converge-attribute-definitions-activerecord-core-readers
  - converge-attribute-definitions-core-readers
  - converge-attribute-definitions-leaf-membership-readers
  - converge-attribute-definitions-peripheral-readers
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "AC1+AC2 delivered by #6948 (retire-attribute-definitions-registry-for-default-attributes, RFC 0115): on origin/main packages/activerecord/src/attributes.ts contains ZERO '_attributeDefinitions' references (git grep -c = 0) — _defaultAttributes():93 now seeds phase 1 from this.columnsHash():96 per attributes.rb:241-245, and defineAttribute routes through defineDefaultAttribute(name, value, type, {fromUser}) per attributes.rb:277-291. The blocked-by's ordering half is also stale: all five reader splits landed (#6769, #6804, #6805, #6806, #6807). The only surviving AC is AC3 (base.ts:1393 ensureSchemaLoaded still iterates this._attributeDefinitions), which is explicitly owned by 0115/retire-remaining-attribute-definitions-registry ('base.ts — ensureSchemaLoaded's declaration-scan bail'), whose body says to re-triage this story against it rather than duplicate it. Closing as superseded."
---

## Context

Residue of `converge-attribute-definitions-activerecord-core-readers` (split 2/4
of [[converge-attribute-definitions-onto-default-attributes]]). That story
converged the AR-core _readers_ — `Base.hasAttribute`,
`inheritance.ts#classHasAttribute` (now `_has_attribute?`),
`isDescendsFromActiveRecord`, `ensureProperType` — onto `attribute_types`, and
dropped the dead `_attributeDefinitions` field from
`attribute-methods.ts`'s host interface. Two clusters were deliberately left,
because they are the map's _owners_, not readers, and converging them means
deleting `_attributeDefinitions` outright:

- `packages/activerecord/src/attributes.ts` — `defineAttribute` (the
  copy-on-write write into the map, `:99-112` on the branch) and
  `_defaultAttributes` (`:171-230`), whose phase-1 seed iterates
  `cacheHost._attributeDefinitions` instead of `columns_hash` alone. Rails seeds
  phase 1 from `columns_hash.transform_values { Attribute.from_database(...) }`
  (`vendor/rails/activerecord/lib/active_record/attributes.rb:241-245`) and lets
  the pending-modification replay supply every non-column declaration; trails
  needs the map only because non-column declarations are not fully carried by
  the pending queue yet.
- `packages/activerecord/src/base.ts#ensureSchemaLoaded` — three readers (the
  `hasOwnProperty` walk to the map's defining class, the `reflectedTable` scan,
  and the `virtual` scan). These read per-definition metadata
  (`reflectedTable`, `virtual`, `source`) that `attribute_types` does not carry.
  `ensureSchemaLoaded` itself has no Rails counterpart — Rails' `attribute_types`
  loads the schema synchronously on first touch — so this converges by deleting
  the method, not by re-pointing its readers.

Also still open (owned by the sibling splits): `model-schema.ts`,
`persistence.ts`, `enum.ts`, `fixtures.ts`, `encryption/*`,
`nested-attributes.ts`, `type-caster/map.ts`.

## Acceptance criteria

- [ ] `attributes.ts#_defaultAttributes` seeds phase 1 from `columnsHash()` alone,
      with non-column declarations arriving through the pending-modification
      replay (Rails `attributes.rb:241-245`).
- [ ] `defineAttribute` stops maintaining `_attributeDefinitions`.
- [ ] `base.ts#ensureSchemaLoaded`'s three `_attributeDefinitions` readers are gone
      (with the method itself, or with the metadata moved to where the schema
      actually records it).
- [ ] `pnpm parity:api:calls` / `:args` green with no new baseline rows.
