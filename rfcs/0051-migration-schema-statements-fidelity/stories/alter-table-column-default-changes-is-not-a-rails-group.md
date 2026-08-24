---
title: "AlterTable's columnDefaultChanges group is a trails invention — route default changes through ChangeColumnDefaultDefinition"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: 30
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `visit_AlterTable`'s `join` separator in PR #6362
(RFC 0099 `call-args-ar-join-separator`).

`AlterTable` in trails carries a `columnDefaultChanges` array
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:933`,
pushed at `:991`) that Rails' `AlterTable` does not have — `grep -rn
"column_default_changes" vendor/rails/activerecord/lib` returns nothing.

`visitAlterTable`
(`packages/activerecord/src/connection-adapters/abstract/schema-creation.ts`)
now mirrors `connection_adapters/abstract/schema_creation.rb:24-32` exactly for
the six Rails groups — each `map{...}.join(" ")` appended to `sql` with no
separator between groups — and then appends a SEVENTH, trails-only group for
`columnDefaultChanges`, joined with `" "`. That extra group is flagged at the
call site with a comment naming `schema_creation.rb:24-32`, which is the
deviation receipt, not permission to keep it.

Rails expresses a default change as a `ChangeColumnDefaultDefinition`
(`connection_adapters/abstract/schema_definitions.rb`), visited by
`visit_ChangeColumnDefaultDefinition` on the PG/MySQL `SchemaCreation`
subclasses and assembled by `change_column_default_for_alter`
(`connection_adapters/abstract/schema_statements.rb`) — never as a group on
`AlterTable`. trails already HAS
`visitChangeColumnDefaultDefinition` on the PG visitor
(`connection-adapters/postgresql/schema-creation.ts`), so the machinery to
converge onto exists.

## Converged shape

Route default changes through `ChangeColumnDefaultDefinition` /
`change_column_default_for_alter` as Rails does, delete the
`columnDefaultChanges` field from `AlterTable`
(`schema-definitions.ts:933`, `:991`) and delete the seventh group and its
deviation comment from `visitAlterTable`, leaving that body a line-for-line
port of `schema_creation.rb:24-32`.

## Acceptance criteria

- [ ] `AlterTable` has no `columnDefaultChanges` field and no pusher.
- [ ] `visitAlterTable` (abstract) is exactly Rails' six groups, no extra
      group and no deviation comment.
- [ ] Every existing caller of the removed pusher reaches the same emitted
      SQL through the `ChangeColumnDefaultDefinition` path.
- [ ] `pnpm parity:api:calls` / `:args` stay green; `pnpm parity:api:extra
--package activerecord` does not grow.
