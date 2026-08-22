---
title: "wave-5d-tail-sweep"
status: claimed
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-22T22:50:07Z"
assignee: "wave-5d-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Continuation of `wave-5c-tail-sweep` (RFC 0106), which took the activesupport
tail plus `activerecord/tasks/**` and `activerecord/database-configurations**`
— 34 rows migrated to `@missingRailsCall` receipts in PR #PR.

What remains of the 2/3-row `kind: "set"` tail for `activerecord` / `arel` /
`activesupport`, measured on 2026-08-22 after that PR: **36 files,
89 rows**.

- `activerecord/associations/alias-tracker.json` — 2 rows
- `activerecord/associations/association-scope.json` — 2 rows
- `activerecord/associations/builder/belongs-to.json` — 2 rows
- `activerecord/associations/collection-association.json` — 3 rows
- `activerecord/associations/has-many-association.json` — 2 rows
- `activerecord/associations/join-dependency.json` — 3 rows
- `activerecord/associations/join-dependency/join-association.json` — 2 rows
- `activerecord/associations/preloader/through-association.json` — 3 rows
- `activerecord/attribute-methods.json` — 3 rows
- `activerecord/attribute-methods/primary-key.json` — 2 rows
- `activerecord/connection-adapters/abstract/schema-creation.json` — 3 rows
- `activerecord/connection-adapters/mysql/column.json` — 2 rows
- `activerecord/connection-adapters/mysql2/database-statements.json` — 2 rows
- `activerecord/connection-adapters/pool-config.json` — 2 rows
- `activerecord/connection-adapters/postgresql/quoting.json` — 3 rows
- `activerecord/connection-adapters/postgresql/schema-dumper.json` — 2 rows
- `activerecord/connection-adapters/sqlite3/database-statements.json` — 2 rows
- `activerecord/connection-adapters/sqlite3/schema-statements.json` — 3 rows
- `activerecord/connection-handling.json` — 2 rows
- `activerecord/core.json` — 3 rows
- `activerecord/delegated-type.json` — 3 rows
- `activerecord/disable-joins-association-relation.json` — 2 rows
- `activerecord/encryption/cipher/aes256-gcm.json` — 2 rows
- `activerecord/encryption/encryptable-record.json` — 3 rows
- `activerecord/encryption/extended-deterministic-queries.json` — 3 rows
- `activerecord/middleware/database-selector/resolver.json` — 3 rows
- `activerecord/migration/command-recorder.json` — 3 rows
- `activerecord/querying.json` — 2 rows
- `activerecord/relation/batches.json` — 3 rows
- `activerecord/relation/where-clause.json` — 3 rows
- `activerecord/scoping/default.json` — 2 rows
- `activerecord/secure-token.json` — 2 rows
- `activerecord/statement-cache.json` — 3 rows
- `activerecord/transactions.json` — 2 rows
- `activerecord/validations/associated.json` — 3 rows
- `activesupport/inflector.json` — 2 rows

Regenerate the list with

    pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls

then group `call-mismatches-exclude/**` by file at `kind: "set"`, keeping the
files with 2 or 3 rows for those three packages.

This is more than one 700 LOC PR (~34 rows is a full PR at ~520 LOC). Ship it
as sequential non-overlapping PRs from `main`, never stacked, and file the
remainder again as a follow-on.

### Rulings carried forward from wave-5c

Three things wave-5c learned the hard way; do not re-derive them:

1. **A migrated tag needs a permanence claim.** `parity:api:build` writes the
   baseline `reason` verbatim, and `suppressedCallsIn`
   (`scripts/api-compare/missing-rails-call-tags.ts:219-230`) then hard-errors
   unless the reason opens with `PERMANENT` or `CONVERGEABLE`. Prefix each
   migrated reason with `PERMANENT:` (plus a space) after the em-dash, then re-run
   `pnpm build`.

2. **A CONVERGEABLE row must NOT migrate.** Per the RFC 0106 rule stated at
   `missing-rails-call-tags.ts:296-299`, a deviation whose work is still owned
   by a story belongs in `call-mismatches-exclude/` as a row, because the row
   is the thing tracking it. Only rows whose reason is a genuine, permanent
   language- or runtime-level fact leave as receipts. wave-5c left five rows
   baselined on this basis (`database-configurations.ts` `call`,
   `database-config.ts` `call`, `mysql-database-tasks.ts` `recreate_database`,
   `number-converter.ts` `merge!`, `values/time-zone.ts` `to_date`).

3. **`activesupport/inflector.json` is a known trap — handle it separately.**
   Its two rows (`safe_constantize` -> `const_regexp` / `match?`) coexist on
   `main` with `@missingRailsCall` tags for the SAME calls on
   `safeConstantize`'s JSDoc. Dropping the rows reports both tags STALE;
   dropping the tags as well then reports both calls as NEW mismatches. The
   tag is not suppressing the flag, so the row and the tag are not
   interchangeable here. wave-5c reverted the file rather than guess. Diagnose
   why the tag on an exported top-level `function` in `inflector.ts` fails to
   suppress before touching this shard.

## Acceptance criteria

- [ ] Every row in the files taken by the PR is converged, or leaves as a
      `@missingRailsCall` tag carrying its reviewed per-site reason, prefixed
      with a `PERMANENT` claim, at the call site. Never a name-keyed bulk edit,
      a broadened reason, or a move to another register.
- [ ] Rows whose reason names a convergence OWNER stay baseline rows.
- [ ] Emptied shards deleted, not committed as `[]`;
      `pnpm parity:api:calls:tighten <shard>` for any shard left with a stale mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] The un-taken remainder is filed as a follow-on story with the current file list.
