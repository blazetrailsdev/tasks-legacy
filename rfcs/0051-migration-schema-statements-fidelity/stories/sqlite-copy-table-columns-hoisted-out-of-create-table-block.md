---
title: "copy_table hoists columns(from) out of the create_table block"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `copy_table` hoists `columns(from)` out of the `create_table` block

## Context

Surfaced converging the sqlite3-adapter call-set rows in PR #6567 (RFC 0106
wave-3a). The row

    activerecord | connection-adapters/sqlite3-adapter.ts | copy_table
    order:columns,createTable -> createTable,columns

is the last `order:` row on that file and could not be converged there; it carries
a reviewed reason in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/sqlite3-adapter.json`.

`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:599-649`:

    def copy_table(from, to, options = {})
      from_primary_key = primary_key(from)
      options[:id] = false
      create_table(to, **options) do |definition|
        @definition = definition
        ...
        columns(from).each do |column|          # <- INSIDE the block
          ...
        end
        yield @definition if block_given?
      end
      ...

Rails reflects `columns(from)` INSIDE the `create_table` block. trails
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts#copyTable`)
hoists it above the `createTable` call:

    const sourceColumns = (await this.columns(from)) as Sqlite3Column[];
    await this.createTable(to, { ...createOptions, id: false }, (td) => { ... });

because `createTable`'s `TableDefinition` block is SYNCHRONOUS in trails — the
definition is built and then rendered with no await point — while `columns()` is
async. Every call Rails makes is made, once, with the same arguments; only the
sequence differs.

## Converged shape

`createTable` awaits its definition block, so the block can perform the async
reflection Rails performs inline, and `copyTable`'s body reads in Rails' order.
Note this changes the block contract for every `createTable` caller (migrations,
schema load, the `alter_table` family), so the signature change is the bulk of the
work, not `copy_table` itself.

## Acceptance criteria

- [ ] `copyTable` calls `columns(from)` inside the `createTable` block, mirroring
      sqlite3_adapter.rb:608.
- [ ] `createTable` awaits a block that returns a promise, and existing
      synchronous blocks keep working unchanged.
- [ ] The `copy_table | order:columns,createTable` row is deleted by hand from its
      shard (no `--write`, no reseed); stale marks fixed with
      `parity:api:calls:tighten`.
- [ ] `alter_table`-driven paths (add/remove/change column, rename) stay green;
      SQLite, PostgreSQL and MySQL/MariaDB lanes green.
