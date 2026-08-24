---
title: "port-with-connection-acquisition-seam-for-the-arel-reader"
status: blocked
updated: 2026-08-24
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: 5
pr: 6928
claim: "2026-08-23T17:39:45Z"
assignee: "converge-excluding-deferred-ids-marker-to-eager-materialization"
blocked-by: "Re-verified against origin/main 2026-08-24: blocker still live. ConnectionPool#withConnection is still `async` (connection-adapters/abstract/connection-pool.ts:961 — the story body's :1008 anchor has drifted), so no synchronous with_connection seam is expressible; a sync seam could only serve an already-leased connection, which is exactly what Relation#_conn() (relation.ts:1142) returns. The reader is now spelled `arel(aliases?): SelectManager` (relation.ts:3381 — the body still calls it `toArel`, renamed since the body was written) and is still synchronous, read inline by 31 non-test call sites. Unblocks only when toSql/arel and their callers go async, or a sync query seam lands — neither fits this RFC's 700 LOC PR ceiling; this wants its own RFC."
closed-reason: null
---

## Context

Rails' `arel` reader acquires through a block:

```ruby
def arel(aliases = nil) # :nodoc:
  @arel ||= with_connection { |c| build_arel(c, aliases) }
end
```

(`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb:1595`)

`converge-toarel-onto-with-connection-acquisition` (PR #6755) landed the
achievable half: both `{ sanitizeLimit }` stand-in objects are gone, and
`Relation#toArel` (`packages/activerecord/src/relation.ts`) now acquires with a
plain `this._conn()` lease — so `buildArel` always gets a real connection and a
model with no established pool raises `ConnectionNotEstablished` where Rails'
`with_connection` would. The connectionless Arel-building callers turned out to
be reachable-in-theory only: `buildFrom`'s duck-typed subquery arm
(`packages/activerecord/src/relation/query-methods.ts`) now calls
`resolved.toArel()` unconditionally, as Rails' `opts.arel` does, and the whole
AR suite is green on it.

What is NOT converged is the acquisition SHAPE. trails' `withConnection`
(`packages/activerecord/src/connection-handling.ts:396`) delegates to
`ConnectionPool#withConnection` (`connection-adapters/abstract/connection-pool.ts:1008`),
which is `async` and therefore `Promise`-returning; `Relation#toArel` is
synchronous and is called synchronously from ~30 sites (`toSql`, `updateAll`,
`deleteAll`, `selectAll`, `buildFrom`, the predicate builder). There is no
synchronous `with_connection` seam to port against, and the `@arel ||=` memo is
absent for the same reason.

## Acceptance criteria

- [ ] A synchronous `withConnection` acquisition seam exists (or `toArel` and
      every caller are made async), so the reader can be
      `with_connection { |c| build_arel(c, aliases) }` verbatim.
- [ ] `Relation#toArel` memoizes as `@arel ||=` does (query_methods.rb:1595).
- [ ] Green on SQLite, PostgreSQL and MySQL/MariaDB.
