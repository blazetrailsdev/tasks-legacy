---
title: "Converge createVirtualTable onto Rails' three-parameter one-liner"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `create_virtual_table` is a three-parameter one-liner, not a dual-signature validator

## Context

Surfaced converging `virtual_tables` in PR #6569 — `createVirtualTable` sits
directly below it in the same file.

`vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:308-314`:

```ruby
# Creates a virtual table
#
# Example:
#   create_virtual_table :emails, :fts5, ['sender', 'title',' body']
def create_virtual_table(table_name, module_name, values)
  exec_query "CREATE VIRTUAL TABLE IF NOT EXISTS #{table_name} USING #{module_name} (#{values.join(", ")})"
end
```

Three required positional parameters, one interpolated `exec_query`, no
validation and no quoting — Rails interpolates `table_name` and `module_name`
raw.

`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts#createVirtualTable`
instead takes `(tableName, optionsOrModuleName?: unknown, values?: unknown)` and
carries invented machinery Rails has none of:

- a dual `(name, options)` / `(name, moduleName, values)` signature, discriminated
  by an `Array.isArray` / `typeof === "object"` sniff, with `moduleName` and
  `values` pulled back out of an options bag;
- a `safeIdent = /^[A-Za-z_][A-Za-z0-9_]*$/` check on the module name that raises
  a bespoke `Error("moduleName must be a valid SQLite identifier")` — an error
  class, message and raise site with no Rails counterpart;
- `quoteTableName(tableName)`, where Rails interpolates the bare name.

## Converged shape

`createVirtualTable(tableName, moduleName, values)` — three required
parameters — emitting
`CREATE VIRTUAL TABLE IF NOT EXISTS ${tableName} USING ${moduleName} (${values.join(", ")})`
through the same call Rails uses. Drop the options-bag arm, the `safeIdent`
guard and its error, and the `quoteTableName` call. Check callers
(`sqlite3/schema-dumper.ts` emits the three-argument form already, as does
`migration/command-recorder.ts`) and migrate any options-bag caller to the
positional form.

## Acceptance criteria

- [ ] `createVirtualTable` mirrors sqlite3_adapter.rb:312-314 — same three
      parameters, same interpolation, same single `exec_query` call.
- [ ] The `safeIdent` check and its bespoke `Error` are gone; no
      `@noRailsEquivalent` replacement is added for them.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new surface;
      SQLite lane green (`virtual_table_test.rb` port), PostgreSQL and
      MySQL/MariaDB lanes green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
