---
title: "wave-5f-head-sweep"
status: done
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6897
claim: "2026-08-22T23:57:28Z"
assignee: "wave-5f-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

RFC 0106's exit condition is **0 rows with `kind: "set"` under
`scripts/api-compare/call-mismatches-exclude/**`for`activerecord`, `arel`and`activesupport`\*\*.

`wave-5-head-sweep` (PR pending) took that population from 447 rows / 157 files
down to **406 rows / 154 files** by migrating the reviewed reasons of four head
files to `@missingRailsCall` tags at the call site:

    connection-adapters/abstract/schema-statements.ts   12
    connection-adapters/sqlite3-adapter.ts              10  (9 migrated, 1 kept)
    connection-adapters/abstract/connection-pool.ts     10
    connection-adapters/abstract/schema-definitions.ts  10

It could not go further inside one 700 LOC PR: a migrated row costs ~7 JSON
deletions plus a JSDoc line, so ~45 rows is a full PR.

The settled disposition procedure, applied per file:

1. `pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls` to refresh the
   artifact.
2. Read each row's reason against the Rails body it cites. **Converge it if it
   is a real divergence** — that is the primary exit.
3. Only a row that is already a reviewed, per-site, Rails-anchored reason may
   leave via `pnpm parity:api:build --package <pkg> --file <tsFile>`, which
   writes the reason as a `@missingRailsCall <ruby_call> — <reason>` tag on the
   declaration and drops the baseline row. Never `--write`, never a reseed.
4. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` must be green. A
   shard left with no rows is **deleted**, not committed as `[]`.

The `order:`-row migration trap wave-5 hit was fixed in PR #6855 —
`applyCallTags` now applies a declaration's tags to every flag its pair raised,
so an `order:` row migrates to a `@missingRailsCall` receipt like any other row.
Do not skip one.

Also note **54 rows repo-wide still carry the RFC 0047/0084 seed placeholder**
(`Baseline (RFC 0084) …pending per-body control-flow convergence review`) and
are concentrated in `store.json` (8), `enum.json` (7), `reflection.json` (6),
`associations.json` (5), `attribute-methods.json` (4) and
`statement-cache.json` (3). Those cannot be migrated — the migrator refuses a
placeholder — so they must be **converged or given a real per-site reason**
first.

## Scope

The story's band was originally the whole 5-row second half plus the 4-row band
(58 rows). It is larger than one PR at the 700 LOC ceiling — a migrated row
costs ~7 JSON deletions plus a JSDoc line — so it ships as sequential
non-overlapping PRs from `main`, never stacked, and this story is **the first
PR's band only**:

    activerecord/connection-adapters/schema-cache.json              5
    activerecord/result.json                                        4
    activesupport/duration.json                                     1 of 4

`values/time-zone.json` (1 row) and `relation/batches.json` (2 rows) were in
this band too, but converged or migrated on `main` while the PR was in review:
PR #6890 made `TimeZone#today` actually call `to_date`, and PR #6898 migrated
the two batches rows. They left the population there, not here.

The remaining rows — the other nine shards, plus the rows above that name a
real divergence rather than a language shortcoming and so must converge rather
than migrate — are `wave-5g-head-sweep`, which lists them per shard with the
specific blocker for each.

## Acceptance criteria

- [ ] Every row in the band above is converged, or leaves as a `@missingRailsCall` tag carrying its reviewed per-site reason at the call site. Never a name-keyed bulk edit, a broadened reason, or a move to another register.
- [ ] Emptied shards deleted, not committed as `[]`; `pnpm parity:api:calls:tighten <shard>` for any shard left with a stale mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] Anything that cannot converge is filed as its own story with the Rails `file:line`, not ratified in place.
