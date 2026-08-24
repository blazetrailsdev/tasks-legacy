---
title: "tableStructureSql splits on \\s* where Rails splits on \\s"
status: closed
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Converged by PR #6295 (3d1b45eee, 'sqlite table_structure_sql'). sqlite3-adapter.ts:2449 on origin/main is `new RegExp(`,(?=\\\\s(?:CONSTRAINT|\"(?:${union})\"))`, \"i\")` — a single \\s, matching sqlite3_adapter.rb:785-786. `git grep 'CONSTRAINT|' origin/main -- packages/activerecord/src` returns only that one line; no \\s* widening remains anywhere."
---

## Context

SQLite's `tableStructureSql` widens Rails' column-split lookahead from a single
`\s` to `\s*`:

```ts
splitter = new RegExp(`,(?=\\s*(?:CONSTRAINT|"(?:${escaped})"))`, "i");
```

Rails splits on exactly one space
(`activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:785-786`):

```ruby
.split(/,(?=\s(?:CONSTRAINT|"(?:#{Regexp.union(column_names).source})"))/i)
```

The in-file comment justifies `\s*` as tolerance for a hand-written multi-line
`CREATE TABLE` (`,\n    "col"`), where the newline+indent defeats the single-`\s`
lookahead — load-bearing for the COLLATE / AUTOINCREMENT / GENERATED enrichment
`columns()` performs (`adapters/sqlite3/collation.test.ts`).

This is a real behavioural divergence, not a formatting nit: `\s*` also matches
**zero** whitespace, so `,"col"` splits in trails and does not in Rails.

Surfaced by #6294, which converged the surrounding method (`query_value(sql,
"SCHEMA")` per rb:775, `quote` per rb:767, and Rails' `unless column_names`
arm per rb:758-761) but deliberately left the splitter alone as out of scope.

## Converged shape

Match rb:785-786's single `\s`. The multi-line tolerance then has to come from
somewhere Rails also has it — establish first whether Rails' own reflection
ever sees a multi-line `CREATE TABLE`. Rails reads `sqlite_master.sql`, which
SQLite stores **verbatim as typed**, so a hand-written multi-line DDL reaches
Rails' splitter too and Rails simply does not split it. If that is so, trails'
`\s*` is compensating for a scenario Rails accepts as unsupported, and the
convergence is to drop the widening and adjust the test that depends on it.

Do not close this by re-justifying `\s*`.

## Acceptance criteria

- [ ] Determine what Rails does with a multi-line `CREATE TABLE` in
      `sqlite_master.sql`; record the finding with a Rails cite.
- [ ] `tableStructureSql`'s splitter matches rb:785-786 (single `\s`), or the
      divergence is shown to be a genuine language/driver shortcoming and is
      cited at the call site.
- [ ] `adapters/sqlite3/collation.test.ts` and the `columns()` enrichment stay
      green.
- [ ] parity:api / parity:test delta non-negative.
