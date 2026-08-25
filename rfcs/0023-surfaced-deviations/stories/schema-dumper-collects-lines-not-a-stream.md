---
title: "SchemaDumper threads a stream, not a lines array (retires 8 naming rows)"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while listing the RFC 0096 `naming` rows for
`packages/activerecord/src/schema-dumper.ts` in PR #6356. Eight of the nine rows
resolve to ONE structural divergence, so renaming locals cannot retire them.

`activerecord/lib/active_record/schema_dumper.rb` threads a STREAM through the
dump: `tables(stream)` calls `table(table_name, stream)` and
`foreign_keys(tbl, foreign_keys_stream)`, each writing into the stream it was
handed. The port collects an array of LINES instead and passes `lines`
everywhere Rails passes `stream` / `foreign_keys_stream`, so the comparator
reports `(ref:tableName, ref:stream)` vs `(ref:tableName, ref:lines)` at every
site.

Remaining rows in the same file that are plain renames and can ride along:
`table` → `columns(table)`, `check_constraints_in_create` → `check_constraints(table)`
and `remove_prefix_and_suffix(table)` (Rails' local is `table`, the port's is
`tableName`); `indexes_in_create` → `index_parts(index)` (port: `idx`);
`format_colspec(value)` (port: `v`).

## Converged shape

A stream object with the `<<` semantics `schema_dumper.rb` relies on (the repo
already ports Ruby IO-ish shapes elsewhere — find the settled one rather than
inventing a second), threaded through `tables` / `table` / `foreign_keys` under
Rails' parameter names.

## Acceptance criteria

1. `tables`, `table` and `foreign_keys` take and write to a stream, matching
   `schema_dumper.rb`'s signatures and parameter names.
2. The plain renames listed above are applied in the same pass.
3. Dumped output is byte-identical to before (the dump is snapshot-tested).
4. `naming` row count for `schema-dumper.ts` drops to 0;
   `pnpm parity:api:calls:args` green.
