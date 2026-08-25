---
rfc: "0102-adapter-version-reader-fidelity"
title: "Adapter version and column-reflection reader fidelity"
status: closed
created: 2026-08-12
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
clusters: []
priority: 2
---

# Adapter version and column-reflection reader fidelity

## Problem

RFC 0072 burned down ~10 stories around `database_version` — the trails-only
warm-ups, the per-adapter `getDatabaseVersion` overrides, the MySQL full-version
and MariaDB memo fields. Three residues survived it, all of the same shape: a
reader on an adapter that answers differently from Rails' because trails made it
synchronous or memoized something Rails does not.

1. `database-version-sync-getter-forces-hand-warms` (blocked) — Rails'
   `database_version` reader (`abstract_adapter.rb:854-856`) self-fetches; trails'
   sync getter cannot, so callers hand-warm the memo. PR #6149 proved the
   fill-by-construction shape insufficient (see the story's blocker for the three
   reproductions); the remaining path is making the version-gated predicates async.
2. `converge-pg-supports-optimizer-hints-memo` — `PostgreSQLAdapter#getDatabaseVersion`
   still eagerly fills `_hasOptimizerHints` with a `pg_available_extensions` query
   Rails does not run there.
3. `mysql-new-column-from-field-folds-on-update-into-default-generated` —
   `newColumnFromField` folds `ON UPDATE` into the `DEFAULT_GENERATED` branch, a
   branch Rails keeps separate.

They belong together because (1) is the root cause the other two were split off
from, and fixing them independently re-opens the same question about what a
version-gated adapter predicate may do synchronously.

## Scope

activerecord adapters only. The execute/raw_execute spine is RFC 0076; adapter
construction is RFC 0094.

## Done means

The three stories land, `database_version` is readable wherever Rails'
`database_version` is with no caller-side warm, and no adapter reader carries a
memo fill Rails does not have.
