---
title: "converge-shard-selector-request-construction-call-set-row"
status: done
updated: 2026-08-25
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 50
priority: null
pr: 7009
claim: "2026-08-24T22:30:08Z"
assignee: "converge-postgresql-database-statements-call-set-rows"
blocked-by: null
closed-reason: null
---

## Context

One residual `kind: "set"` row on
`scripts/api-compare/call-mismatches-exclude/activerecord/middleware/shard-selector.json`:
`call -> new`.

`ActiveRecord::Middleware::ShardSelector#call` wraps the rack env in
`ActionDispatch::Request.new(env)` (`shard_selector.rb:41`) before handing it to
the resolver. trails has no `ActionDispatch::Request`, so `call()` is handed the
request itself and constructs nothing — the resolver block therefore receives a
different object than Rails' block does, and any resolver written against Rails'
`request.params` / `request.headers` surface does not port across.

This is an unported-dependency row: convergeable, but only once
`ActionDispatch::Request` exists. It had no owning RFC and no story, so it is
filed here to keep it claimable rather than invisible.

## Acceptance criteria

- [ ] Either `call` constructs an `ActionDispatch::Request` from the env the way
      Rails does, or the row carries a CONVERGEABLE `@missingRailsCall` receipt
      naming the actionpack port this blocks on plus this story slug.
- [ ] The row is deleted from the exclude tree by hand only when it converges;
      no `--write`, no reseed, no widened allowlist.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
