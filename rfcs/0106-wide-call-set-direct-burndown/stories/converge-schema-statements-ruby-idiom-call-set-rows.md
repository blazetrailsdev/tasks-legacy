---
title: "converge-schema-statements-ruby-idiom-call-set-rows"
status: in-progress
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7009
claim: "2026-08-24T22:30:08Z"
assignee: "converge-postgresql-database-statements-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

Two residual `kind: "set"` rows on
`scripts/api-compare/call-mismatches-exclude/activerecord/connection-adapters/abstract/schema-statements.json`
are Ruby-idiom conversions delegated to RFC 0082
(`0082-ruby-ts-idiom-conversion-classes`), which is `postponed` and has no story
naming them:

| ruby method           | missing call | Rails                            | idiom class                       |
| --------------------- | ------------ | -------------------------------- | --------------------------------- |
| `generate_index_name` | `limit`      | `schema_statements.rb:1606-1609` | `String#limit` (byte truncation)  |
| `index_algorithm`     | `fetch`      | `schema_statements.rb:1504-1508` | `Hash#fetch` with a raising block |

`generate_index_name` truncates the identifier with ActiveSupport's
`String#limit`, which counts **bytes**; the port truncates by JS UTF-16 length,
so a multi-byte table or column name yields a different index name than Rails
on the same schema. `index_algorithm` is
`index_algorithms.fetch(algorithm) { raise ArgumentError, ... }`; the port
throws inline off a `Map` lookup.

## Acceptance criteria

- [ ] `@blazetrails/activesupport` exposes the `String#limit` byte-truncation
      idiom (or the existing one is used) and `generateIndexName` calls it.
- [ ] `indexAlgorithm` goes through the ported `Hash#fetch`-with-block idiom,
      raising the same `ArgumentError` with the same message.
- [ ] Both rows deleted by hand; `pnpm parity:api:calls` / `:args` green; no
      `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
