---
title: "dbconsole never reads ActiveRecord.databaseCli (find_cmd_and_exec unported)"
status: draft
updated: 2026-07-29
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

Surfaced by #5564 (RFC 0081, `convert-ar-config-accessors-internal-flags`).
Converting `databaseCli` to an accessor made it a ported method, so the wide
calls ratchet started checking the Rails bodies that read it and flagged all
three `dbconsole` implementations.

Rails reads the cli name and hands it straight to `find_cmd_and_exec`:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/sqlite3_adapter.rb:44-52`
  — `find_cmd_and_exec(ActiveRecord.database_cli[:sqlite], *args)`
- the postgresql and abstract_mysql adapters do the same with
  `database_cli[:postgresql]` / `database_cli[:mysql]`

Trails ports only the argument construction: `dbconsole` returns `string[]` and
never consults `ActiveRecord.databaseCli`
(`packages/activerecord/src/connection-adapters/sqlite3-adapter.ts:1509-1520`,
plus postgresql-adapter.ts and abstract-mysql-adapter.ts). The PTY exec itself
is Ruby-only, which is why the lookup was dropped.

The three mismatches are baselined in
`scripts/api-compare/call-mismatches-wide-exclude/activerecord/` with that
reason. This story is to decide the real shape: either return the resolved cli
command alongside the args (so `databaseCli` is actually honored and a caller
could exec it), or confirm the drop is correct and record it as a permanent
deviation rather than a ratchet baseline entry.

Note the sibling story `mysql-dbconsole-database-pushed-unconditionally`
(0023) touches the same mysql method — sequence them.

## Acceptance criteria

- A decision recorded for whether trails' `dbconsole` should resolve
  `ActiveRecord.databaseCli`, with the Rails `file:line` backing it.
- If yes: all three adapters read the flag, and the three
  `dbconsole`/`database_cli` entries are removed from
  `call-mismatches-wide-exclude/`.
- If no: the exclude entries' reason is upgraded to point at the deviation
  record instead of restating the unported-exec rationale.

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
