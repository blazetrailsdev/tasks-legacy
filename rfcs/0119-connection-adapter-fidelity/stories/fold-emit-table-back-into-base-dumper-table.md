---
title: "fold-emit-table-back-into-base-dumper-table"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

`emitTable` and `emitTableBody` live in
`packages/activerecord/src/connection-adapters/abstract/schema-dumper.ts`, but
they are the column-emission half of Rails' **base** `SchemaDumper#table`
(`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:150-230`) — Rails
has no `emit_table` / `emit_table_body` at all, in either class. The abstract
adapter file (`connection_adapters/abstract/schema_dumper.rb`) contains only
`create` and the private column-spec helpers.

The split was a consequence of the mixin-host arrangement PR #6140 removed: the
bodies had to sit in the adapter file to keep the parity:api file mapping while
the base owned the single class. With a real subclass in place, that constraint
is gone — the decomposition is now pure invented surface in the wrong file.

## Converged shape

One `table` on the base, matching `schema_dumper.rb:150-230`'s decomposition:
the local-buffer/rescue structure Rails uses (`tbl` StringIO, "Could not dump
table" on raise) stays, and no `emit_table` / `emit_table_body` pair exists. The
adapter file keeps only `create`, `DEFAULT_DATETIME_PRECISION` and the
column-spec helpers, as the Ruby file does.

## Acceptance criteria

- [ ] `emitTable` / `emitTableBody` are gone; their logic lives in the base's
      `table`, decomposed as Rails decomposes it.
- [ ] `connection-adapters/abstract/schema-dumper.ts` mirrors the members of
      `connection_adapters/abstract/schema_dumper.rb` and nothing else.
- [ ] `pnpm parity:api:extra --package activerecord` drops the two names; dumper suites
      green on all three lanes.
