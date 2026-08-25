---
title: "copy_table_contents adds an empty-columns early return Rails does not have"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`SQLite3Adapter#copy_table_contents` has no empty-column guard
(`activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:679-690`):
it filters `columns`, builds the quoted lists, and issues the INSERT
unconditionally.

The port (`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts`,
`copyTableContents`) adds an early return Rails does not have:

```ts
columns = columns.filter((col) => fromColumns.includes(columnMappings[col]));
if (!columns.length) return;
```

PR #6353 converged the identifiers in this method to Rails' (`column_mappings`,
`from_columns_to_copy`, `quoted_columns`, `quoted_from_columns`, the `col` block
var) but deliberately left the extra guard, since removing a branch is a
behaviour change outside a rename story.

## Converged shape

Drop the guard so the method mirrors sqlite3_adapter.rb:679-690 exactly. Rails
reaching this with an empty column list emits
`INSERT INTO "t" () SELECT  FROM "f"` and lets SQLite raise; the port silently
succeeds, which hides an alter_table bug rather than surfacing it.

Establish first whether any live `alter_table` path can reach it empty — if one
can, the fix is at that caller (Rails would never call `copy_table_contents`
with no columns), not a guard inside the ported body.

## Acceptance criteria

1. `copyTableContents` has no early return Rails lacks.
2. If a caller could reach it with an empty list, that caller is fixed with the
   Rails citation for its shape.
3. The sqlite3 copy-table tests still pass, including the ones that drive
   `alterTable` through add/remove/rename column.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
