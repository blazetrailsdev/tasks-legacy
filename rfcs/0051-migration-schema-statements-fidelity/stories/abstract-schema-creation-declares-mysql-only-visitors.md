---
title: "Abstract SchemaCreation declares indexInCreate/visitIndexDefinition that Rails puts only on MySQL"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 90
priority: 31
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6100 added two members to
`packages/activerecord/src/connection-adapters/abstract/schema-creation.ts`
that Rails' abstract `SchemaCreation` does not declare:

- `indexInCreate(tableName, columnName, options)`
- `visitIndexDefinition(o, create = false)`

Both raise `NotImplementedError` on the abstract visitor. Rails defines each
only on `MySQL::SchemaCreation`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql/schema_creation.rb:40,98`);
the base gets away with it because `accept` dispatches through
`send("visit_#{o.class.name.split('::').last}", o)`
(abstract/schema_creation.rb:12-14) and
`visit_TableDefinition`'s call to `index_in_create`
(abstract/schema_creation.rb:52-55) is gated on `supports_indexes_in_create?`,
false everywhere but MySQL. Ruby resolves the name at call time; TS's
`instanceof` chain and static typing need the member to exist on the receiver
type.

They are counted as extra surface today (`pnpm parity:api:extra --package activerecord`
lists both under `connection-adapters/abstract/schema-creation.ts`).

## Converged shape

Remove both from the abstract class. Candidate approach: type `accept`'s
dispatch table so the `IndexDefinition` branch and the inline-index branch
resolve against a visitor interface the MySQL subclass satisfies (e.g. a
`this`-typed narrowing or a protected optional member checked at the branch),
so the abstract file declares no method Rails does not have while
`MySQL::SchemaCreation` keeps both at their Rails names and positions. If no
shape removes them, `pnpm tasks block` with the specific TS limitation.

## Acceptance criteria

- `abstract/schema-creation.ts` declares neither `indexInCreate` nor
  `visitIndexDefinition`.
- MySQL's inline-index path (`visit_TableDefinition` →
  `index_in_create` → `accept` → `visit_IndexDefinition`) still renders the same
  SQL; `packages/activerecord/src/connection-adapters/{abstract,mysql}/schema-creation.test.ts`
  stay green.
- `pnpm parity:api:extra --package activerecord` drops both names from the abstract file.
