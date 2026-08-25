---
title: "clear_cache! mutates the statement pool outside the connection lock on mysql2/sqlite3"
status: claimed
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 60
pr: null
claim: "2026-08-25T16:42:34Z"
assignee: "converge-clear-cache-lock-mysql-sqlite"
blocked-by: null
closed-reason: null
---

## Context

Rails puts `@lock.synchronize` around the statement-pool mutation in
`AbstractAdapter#clear_cache!` (`abstract_adapter.rb:741-747`), so every adapter
inherits it. trails has no locked abstract `clearCacheBang` — the hook at
`packages/activerecord/src/connection-adapters/abstract-adapter.ts:1475` is an
empty no-op and each adapter overrides it wholesale.

PR #6606 added the lock to the PostgreSQL override only, because that is where
the missing lock was actually observable: PG's eviction `DEALLOCATE`s went onto
the wire outside the lock, left `_commandSettled` false, and made
`execRollbackDbTransaction` read `PQTRANS_ACTIVE` where Rails reads
`PQTRANS_INTRANS` — firing a CancelRequest that landed on a later query
(the recurring PG-shard `Unhandled Rejection: QueryCanceled`).

The other two overrides still mutate their pools unlocked:

- `connection-adapters/mysql2-adapter.ts:355-362` — `void this._statementPool?.clear()`,
  whose `dealloc` sends `DEALLOCATE PREPARE`.
- `connection-adapters/sqlite3-adapter.ts:1413-1420` — `void this._statementPool.clear()`.

## Acceptance criteria

- The statement-pool mutation in the mysql2 and sqlite3 `clearCacheBang`
  overrides runs under the adapter's connection lock, mirroring
  `abstract_adapter.rb:741-747`.
- Prefer converging the shape onto ONE locked `clearCacheBang` if the
  adapter-specific state (`_needsDeallocateAll`, `_rawConnection`) can be
  expressed without a per-adapter override — Rails has no adapter override of
  `clear_cache!` at all.
- No behaviour change to the PG path landed in #6606.
- MySQL/MariaDB and SQLite legs green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
