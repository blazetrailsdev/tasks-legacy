---
title: "wave-5c-tail-sweep"
status: done
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6882
claim: "2026-08-22T20:50:05Z"
assignee: "wave-5c-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Continuation of `wave-5-tail-sweep`. That story's scope was every shard under
`scripts/api-compare/call-mismatches-exclude/**` carrying 3 or fewer
`kind: "set"` rows for `activerecord` / `arel` / `activesupport`. The
single-row shards were taken by `wave-5-tail-sweep` (activerecord) and
`wave-5b-tail-sweep` (activesupport).

What remains in the tail after those two land: the **2-row and 3-row shards** —
25 files with 2 rows and 23 files with 3 rows, 119 rows total as measured on
2026-08-21. Regenerate the current list with

    pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls

then group `call-mismatches-exclude/**` by file at `kind: "set"`, keeping the
files with 2 or 3 rows for those three packages.

This is more than one 700 LOC PR (a migrated row costs ~7 JSON deletions plus a
multi-line JSDoc tag, so ~45 rows is a full PR). Ship it as sequential
non-overlapping PRs from `main`, never stacked, and file the remainder as
follow-on stories.

## Scope and procedure

Per file, in order:

1. Read each row's reason against the Rails body it cites. **Converge it if it
   is a real divergence** — that is the primary exit.
2. A row that is already a reviewed, per-site, Rails-anchored reason may leave
   via `pnpm parity:api:build --package <pkg> --file <tsFile> --call <ruby_call>`.
   Always pass `--call` so the migration is per-row.
3. A row whose reason is still the RFC 0047/0084 seed placeholder
   (`Baseline (RFC 0047)… pending per-cluster burndown review`, `Baseline (RFC
0084): ORDER-only divergence…`) **cannot be migrated** — the migrator refuses
   a placeholder. Converge it, or give it a real per-site reason first.
4. `order:` rows must stay baseline rows: the flag is keyed on the **Ruby** name
   (`copy_table`) while `parity:api:build` writes the tag keyed on the camelCased
   declaration (`copyTable`), so a migrated `order:` row reads back as a STALE
   tag AND a NEW mismatch.

## Acceptance criteria

- [ ] Every row in the files taken by this PR is converged, or leaves as a `@missingRailsCall` tag carrying its reviewed per-site reason at the call site. Never a name-keyed bulk edit, a broadened reason, or a move to another register.
- [ ] Emptied shards deleted, not committed as `[]`; `pnpm parity:api:calls:tighten <shard>` for any shard left with a stale mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] The un-taken remainder of the 2/3-row tail is filed as a follow-on story with the current file list.
