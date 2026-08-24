---
title: "converge-sqlite3-adapter-construction-call-set-rows"
status: claimed
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: "2026-08-24T22:30:08Z"
assignee: "converge-postgresql-database-statements-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

Five residual `kind: "set"` rows in
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/sqlite3-adapter.json`
delegate to RFC 0094 (sqlite3 adapter construction fidelity). 0094's stories
(`sqlite3-connection-parameters-never-built`,
`sqlite-pragmas-option-validation-diverges-from-rails`, …) describe the same
defect but none of them names these rows, so retiring them is nobody's exit
criterion. Filed here per RFC 0106's charter.

| ruby method            | missing call | Rails                        |
| ---------------------- | ------------ | ---------------------------- |
| `initialize`           | `merge`      | `sqlite3_adapter.rb:128-132` |
| `connect`              | `new_client` | `sqlite3_adapter.rb:807`     |
| `new_client`           | `include?`   | `sqlite3_adapter.rb:36-41`   |
| `configure_connection` | `fetch`      | `sqlite3_adapter.rb:837`     |
| `shared_cache?`        | `fetch`      | `sqlite3_adapter.rb:473`     |

All five trace to one root cause: trails' SQLite adapter constructor takes
`(filename, options)` rather than holding a `@connection_parameters` hash built
by merging `@config`, so there is nothing to `merge` into, nothing to hand
`new_client`, and no `flags`/`pragmas` entries to `fetch` from. Converging the
construction shape (0094's
`sqlite3-connection-parameters-never-built` +
`retire-sqlite3-positional-constructor-overload`) retires four of the five
mechanically; `new_client -> include?` needs the `Errno::ENOENT` classification
moved back to open time from `translateException`.

## Acceptance criteria

- [ ] Each row is converged, or carries an honestly classified
      `@missingRailsCall` receipt naming the specific TypeScript shortcoming.
- [ ] If the work lands as part of RFC 0094's construction convergence instead,
      this story is closed with that PR and the rows are deleted there — it must
      not be closed while any of the five rows survives.
- [ ] `pnpm parity:api:calls` / `:args` green; no `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
