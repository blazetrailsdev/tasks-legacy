---
title: "sqlite3-read-uncommitted-shared-cache-skip"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
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
closed-reason: null
---

## Context

`packages/activerecord/src/adapters/sqlite3/transaction.test.ts:78` is an
`it.skip` for Rails'
`vendor/rails/activerecord/test/cases/adapters/sqlite3/transaction_test.rb:42`
`test "opens a \`read_uncommitted\` transaction"`.

Rails opens two connections with `shared_cache_flags`
(`transaction_test.rb:43,49` → `SQLite3::Constants::Open::SHAREDCACHE`), so
`conn2` in a `read_uncommitted` transaction sees `conn1`'s uncommitted INSERT.
trails' driver is better-sqlite3, which does not expose
`SQLITE_OPEN_SHAREDCACHE` on its open call, so two trails connections cannot
share a cache and the assertion cannot hold.

The skip is recorded in `scripts/parity/unported-files/baseline.json:112-116`
("better-sqlite3 does not expose this flag"). It previously also carried a
`PERMANENT-SKIP: driver-limit` comment at the call site; the free-form comment
sweep (`strip-freeform-comments-ar-adapters`, PR #6945) deleted that comment,
per the sweep's rule that a known-divergent shape becomes a story rather than a
better comment. This is that story.

Note the sibling test at `transaction.test.ts:65`
(`raises when trying to open a read_uncommitted transaction but shared-cache
mode is turned off`, `transaction_test.rb:30`) DOES pass — only the
shared-cache-positive arm is skipped.

## Acceptance criteria

- [ ] Determine whether the shared-cache open flag is reachable from trails'
      SQLite driver surface at all (better-sqlite3 open options, or a
      `PRAGMA`/URI route that better-sqlite3 honours — note it does not set
      `SQLITE_OPEN_URI`, so a `file::memory:?cache=shared` filename is opened
      literally).
- [ ] If reachable: unskip `transaction.test.ts:78` and make it pass against
      the Rails assertions verbatim; drop the row at
      `scripts/parity/unported-files/baseline.json:112-116`.
- [ ] If genuinely unreachable: `pnpm tasks block` this story with the driver
      API evidence, leaving the baseline row as the single register.
- [ ] Do NOT close this by re-adding a call-site comment — the sweep's glob
      (`packages/activerecord/src/adapters/**/*.ts` in `eslint.config.mjs`)
      forbids it.
