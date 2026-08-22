---
title: "struct-member-missing-rows-in-activerecord"
status: done
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6879
claim: "2026-08-22T20:19:57Z"
assignee: "struct-member-missing-rows-in-activerecord"
blocked-by: null
closed-reason: null
---

## Context

`extract-ruby-api.rb` now synthesizes the reader/writer/`initialize` methods a
`Struct.new(:a, :b)` superclass generates (RFC 0117,
`struct-members-not-extracted-as-ruby-methods`). That surfaced 12 previously
invisible missing-method rows in `activerecord`:

- `connection_adapters/postgresql/schema_definitions.rb` — 8 rows on
  `ExclusionConstraintDefinition` (`table_name`, `table_name=`, `expression`,
  `expression=`, `options`, `options=`) and `UniqueConstraintDefinition`
  (`column`, `column=`), from
  `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb`.
- `migration.rb` — `reverting` / `reverting=` on
  `ActiveRecord::Migration::ReversibleBlockHelper`.
- `associations/belongs_to_association.rb` — `primary_key`.
- `associations/preloader/batch.rb` — `loaders`.

Two distinct causes:

1. **TS constructor parameter properties are not extracted as members.**
   `packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:108-113`
   spells the Struct members as `constructor(readonly tableName: string, ...)`.
   `extract-ts-api.ts` never walks `ts.ParameterPropertyDeclaration`, so those
   readers look absent. This is a TS-extractor gap, not a port gap.
2. **The writer half is genuinely absent** where the TS field is `readonly`.

## Acceptance criteria

- `extract-ts-api.ts` records a constructor parameter property (`readonly x` /
  `private x` / `public x`) as a member of the class, at the right visibility.
- Decide and implement the writer half: either the ports gain the `setX()`
  writers Rails' Struct generates, or the writer synthesis is scoped where a
  Ruby Struct member is never written.
- `pnpm parity:api` missing-method count for `activerecord` returns to its
  pre-RFC-0117 value (2 for the data layer).
- `pnpm vitest run scripts/api-compare` green.
