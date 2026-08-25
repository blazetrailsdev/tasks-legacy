---
title: "arel hosts PostgreSQL::Utils.extract_schema_qualified_name (postgresql/utils.rb:60)"
status: ready
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while burning down arel's `@internal` tags for RFC 0121 (#7045).

`packages/arel/src/visitors/split-schema-qualified-name.ts` implements two
helpers that have no arel counterpart in Rails:

- `splitSchemaQualifiedName(name)` — splits on unquoted dots, preserving
  double-quoted segments.
- `quoteSchemaQualifiedName(name)` — re-quotes each part with ANSI double
  quotes.

Rails puts both in **activerecord**, not arel:

- `ConnectionAdapters::PostgreSQL::Utils.extract_schema_qualified_name`
  (`activerecord/lib/active_record/connection_adapters/postgresql/utils.rb:60`)
- `PostgreSQL::Utils::Name#quoted` (`postgresql/utils.rb:22`), reached through
  `PostgreSQL::Quoting#quote_table_name`
  (`postgresql/quoting.rb:58-59`, `QUOTED_TABLE_NAMES[name] ||= Utils.extract_schema_qualified_name(name.to_s).quoted`)

They live in arel today only because `Visitors::ToSql` needs the behaviour and
arel must not depend on activerecord. Both currently carry `@internal` plus a
`@noRailsEquivalent CONVERGEABLE` receipt pointing here.

Consumers today: `packages/arel/src/visitors/index.ts`,
`packages/arel/src/test-helpers/default-quoter.ts`, and
`packages/activerecord/src/connection-adapters/postgresql/quoting.ts` — that
last one is activerecord reaching INTO arel for logic Rails already owns in
activerecord, which is the dependency inversion to undo.

## Converged shape

- Port `PostgreSQL::Utils` (the `Name` struct + `extract_schema_qualified_name`)
  into activerecord at the Rails path
  (`connection-adapters/postgresql/utils.ts`), with `Name#quoted`.
- Have `postgresql/quoting.ts` call it directly rather than importing from arel.
- arel reaches the behaviour the way Rails does — through the connection
  (`ArelConnection#quoteTableName`), not by hosting the helper.
- Delete `split-schema-qualified-name.ts` and its two receipts.

## Acceptance criteria

- `PostgreSQL::Utils.extract_schema_qualified_name` and `Name#quoted` exist in
  activerecord at the Rails-layout path, mirroring `postgresql/utils.rb:22,60`.
- No activerecord file imports schema-qualified-name helpers from arel.
- `packages/arel/src/visitors/split-schema-qualified-name.ts` deleted, both
  `@noRailsEquivalent` receipts gone.
- `pnpm parity:api:extra` reports no STALE tag and `pnpm parity:api:extra:gate`
  is green (arel's `total` should DROP; tighten the mark with
  `pnpm parity:api:extra:tighten`, never raise it).
