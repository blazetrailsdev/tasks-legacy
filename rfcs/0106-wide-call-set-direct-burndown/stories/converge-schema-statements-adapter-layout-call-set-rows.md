---
title: "converge-schema-statements-adapter-layout-call-set-rows"
status: done
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: 7008
claim: "2026-08-24T22:18:08Z"
assignee: "converge-delegated-type-and-default-scope-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

Three residual `kind: "set"` rows on
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract/schema-statements.json`:

| ruby method   | missing call            | Rails                          |
| ------------- | ----------------------- | ------------------------------ |
| `columns`     | `column_definitions`    | `schema_statements.rb:107-113` |
| `columns`     | `new_column_from_field` | `schema_statements.rb:107-113` |
| `primary_key` | `primary_keys`          | `schema_statements.rb:145-149` |

Rails' abstract `columns` is
`column_definitions(table_name).map { |field| new_column_from_field(table_name, field, definitions) }`
— both callees are per-adapter overrides. trails' `columns` instead switches on
`adapterName` inside the abstract body, so there is no seam to call; same for
`primary_key`, which switches rather than delegating to a per-adapter
`primaryKeys`.

This is the adapter-layout deviation RFC 0023 tracks generally, but no 0023
story names these rows and RFC 0023 is `postponed`, so they are filed here.
Related: `project_fidelity_rfcs_adapter_layout_join_dependency`.

## Acceptance criteria

- [ ] The abstract `columns` calls `columnDefinitions` and `newColumnFromField`,
      with the per-adapter bodies living on the PostgreSQL / MySQL / SQLite
      adapters where Rails puts them; the `adapterName` switch is deleted.
- [ ] `primaryKey` delegates to a per-adapter `primaryKeys` the same way.
- [ ] The three rows are deleted from the exclude tree by hand; stale marks
      narrowed with `pnpm parity:api:calls:tighten`. No `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
