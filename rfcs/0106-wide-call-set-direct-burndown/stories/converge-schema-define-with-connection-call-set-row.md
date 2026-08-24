---
title: "converge-schema-define-with-connection-call-set-row"
status: done
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 7008
claim: "2026-08-24T22:18:08Z"
assignee: "converge-delegated-type-and-default-scope-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

One residual `kind: "set"` row on
`scripts/api-compare/call-mismatches-exclude/activerecord/schema.json`:
`define -> with_connection`.

`ActiveRecord::Schema.define` is
`connection_pool.with_connection { |connection| ... }` (`schema.rb:54`) — the
block scopes a checkout for the whole schema definition. trails' `Schema.define`
is handed the adapter explicitly (`packages/activerecord/src/schema.ts:51-64`)
and checks nothing out.

The row's reason delegates this to RFC 0059 (one-schema convergence), which is
**closed** — so the row was left with no live owner. Refiled here.

## Acceptance criteria

- [ ] `Schema.define` acquires its adapter through `connectionPool.withConnection`
      the way Rails does, rather than taking one as a parameter — or carries an
      honestly classified `@missingRailsCall` receipt naming the specific
      TypeScript shortcoming (the sync/async lease split of RFC 0073) and a live
      successor story.
- [ ] The row is deleted from the exclude tree by hand; no `--write`, no reseed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
