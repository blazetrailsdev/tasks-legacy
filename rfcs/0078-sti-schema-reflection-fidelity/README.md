---
rfc: "0078-sti-schema-reflection-fidelity"
title: "STI / schema-reflection attribute-definition fidelity"
status: superseded
created: 2026-07-26
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
  - "activemodel"
clusters:
  - "schema"
priority: 2
superseded-by: "0123-blocked-convergence-holding"
---

Extracted from RFC 0023 (surfaced-deviations) triage, 2026-07-26.

trails' `_attributeDefinitions` + STI overlay + reflection-registry generation gate is invented machinery standing in for Rails' `_default_attributes` + `reload_schema_from_cache` + Zeitwerk discard semantics. The open stories here (cold-leaf STI gates, the unenforced ownSchemaMemo/\_schemaRevision stale invariant, declared-attribute default seeding, enum decorator replay reentrancy, registry poison mechanism) all trace to that same substitution and should converge together - the headline story is converge-attribute-definitions-onto-default-attributes.

Swept 2026-08-09 against origin/main: 5 of 14 stories closed as already converged (STI overlay machinery, STI base attribute routing, the split bound schema cache, reload_schema_from_cache's STI apparatus, descendant invalidation, and the subclass-tableName clobber repro). 9 remain.
