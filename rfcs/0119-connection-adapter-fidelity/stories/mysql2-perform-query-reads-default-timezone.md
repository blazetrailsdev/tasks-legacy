---
title: "mysql2 performQuery does not read default_timezone; callers sync it instead"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Found while relocating `ActiveRecord.default_timezone` (PR #5566).

Rails re-reads the timezone _inside_ `perform_query`:
`vendor/rails/activerecord/lib/active_record/connection_adapters/mysql2/database_statements.rb:47-49`
— `raw_connection.query_options[:database_timezone] = default_timezone`, with a
comment saying the point is to carry over changes made since the connection was
established.

trails' `packages/activerecord/src/connection-adapters/mysql2/database-statements.ts`
`performQuery` never reads it. Instead each caller in
`packages/activerecord/src/connection-adapters/mysql2-adapter.ts` calls the
trails-only `_syncDatabaseTimezone()` one frame up: `internalExecQuery` (:672),
`execute` (:1041), `executeMutation` (:1092), `internalExecute` (:1270),
`explain` (:1394). Any future caller that forgets the call silently keeps a
stale `databaseTimezone`.

This is baselined as a wide call-mismatch in
`scripts/api-compare/call-mismatches-wide-exclude/activerecord/connection-adapters/mysql2/database-statements.json`
(`perform_query` -> `default_timezone`), and the sibling entry in
`.../mysql2-adapter.json` (`configure_connection` -> `default_timezone`) covers
the same divergence from `mysql2_adapter.rb:160`.

## Acceptance criteria

- `performQuery` sets `databaseTimezone` from `ActiveRecord.defaultTimezone`
  itself, at the Rails position (before the query runs).
- `_syncDatabaseTimezone()` calls in the five `mysql2-adapter.ts` callers are
  removed; the helper itself goes away if nothing else needs it.
- `configure_connection` sets it at connection time too, matching
  `mysql2_adapter.rb:160`.
- The two `default_timezone` wide-call baseline entries above are deleted, not
  reworded, and `pnpm exec tsx scripts/api-compare/lint-call-mismatches-wide.ts`
  stays green.
- A test proves a `withTimezoneConfig({ default: "local" })` change is observed
  by the very next query without any adapter-level sync call.

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
