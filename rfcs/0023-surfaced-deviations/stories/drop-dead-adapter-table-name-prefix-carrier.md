---
title: "Remove or document the unpopulated adapter tableNamePrefix/Suffix carrier"
status: draft
updated: 2026-07-28
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Three sites in the connection-adapter layer read the table-name prefix/suffix as
`adapter.tableNamePrefix ?? globalTableNamePrefix()`:

- `TableDefinition#newForeignKeyDefinition`
  (`connection-adapters/abstract/schema-definitions.ts`)
- `TableDefinition#_foreignKeyOptions`'s `columnFor` fallback (same file)
- `SchemaStatements#stripTableNamePrefixAndSuffix`
  (`connection-adapters/abstract/schema-statements.ts:2420-2423`)

Rails reads `ActiveRecord::Base.table_name_prefix` / `.table_name_suffix`
unconditionally at the equivalent sites (`schema_definitions.rb:575-576`,
`schema_statements.rb:1750-1751`). There is no adapter-level override in Rails.

The `adapter.tableNamePrefix` left operand is a trails-only carrier that
**nothing in the library populates** — `ForeignKeyOptionsAdapter` declares the
fields optional and only tests
(`schema-definitions.test.ts`, `schema-statements-privates.test.ts`) ever set
them. So in production the `??` always falls through to the registry, and
behavior is equivalent to Rails today. The hazard is that a reader cannot tell
where the field is set, and an adapter that ever sets it would silently diverge
from Rails.

Surfaced during review of #5488 (reviewer flagged it twice as worth documenting;
declined there because the PR was under a no-comments instruction and it was
explicitly not a finding).

## Acceptance criteria

- [ ] Either drop the `adapter.tableNamePrefix`/`tableNameSuffix` carrier and
      read the `table-name-options` registry unconditionally at all three sites
      (matching Rails), migrating the tests that currently set it, **or** keep it
      and document at the declaration why it exists and who may set it.
- [ ] If dropped: `ForeignKeyOptionsAdapter`'s `tableNamePrefix`/
      `tableNameSuffix` members go away and the tests that populate them set
      `Base.tableNamePrefix` instead.
- [ ] No behavior change for the default (unset) case; existing prefix/suffix
      tests stay green.
