---
title: "wave-5e-tail-sweep"
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
claim: "2026-08-22T23:42:34Z"
assignee: "wave-5e-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Continuation of `wave-5d-tail-sweep` (RFC 0106), which took the eight
`activerecord/associations/**` shards — 16 rows migrated to `@missingRailsCall`
receipts, five shards deleted.

What remains of the 2/3-row `kind: "set"` tail for `activerecord` / `arel` /
`activesupport`, measured after that PR: **28 files, 70 rows**.

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

This is more than one 700 LOC PR. Ship it as sequential non-overlapping PRs
from `main`, never stacked, and file the remainder again as a follow-on.

### Rulings carried forward from waves 5c/5d

1. **A migrated tag needs a permanence claim.** `parity:api:build` writes the
   baseline `reason` verbatim, and `suppressedCallsIn`
   (`scripts/api-compare/missing-rails-call-tags.ts:219-230`) hard-errors unless
   the reason opens with `PERMANENT` or `CONVERGEABLE`. Prefix each migrated
   reason with `PERMANENT:` in the baseline JSON first (through
   `serializeBaseline`, never a hand-edit), then run
   `pnpm parity:api:build --package <pkg> --file <tsFile> --call <ruby_call>`.

2. **A CONVERGEABLE row must NOT migrate.** Per the RFC 0106 rule at
   `missing-rails-call-tags.ts:296-299`, a deviation whose work is still owned
   belongs in `call-mismatches-exclude/` as a row. Wave 5d left three rows
   baselined on this basis: `collection-association.json` `transaction` (the
   sync writer's `ReplacePlan` split), `join-dependency.json` `match?`
   (`aliasSet.has` in place of Rails' alias regex), and
   `preloader/through-association.json` `scope` (the source Preloader is handed
   a derived scope).

3. **`activesupport/inflector.json` is a known trap — handle it separately.**
   Its two rows (`safe_constantize` -> `const_regexp` / `match?`) coexist on
   `main` with `@missingRailsCall` tags for the SAME calls on
   `safeConstantize`'s JSDoc. Dropping the rows reports both tags STALE;
   dropping the tags as well then reports both calls as NEW mismatches. The tag
   is not suppressing the flag, so the row and the tag are not interchangeable
   here. Waves 5c and 5d both reverted rather than guess. Diagnose why the tag
   on an exported top-level `function` in `inflector.ts` fails to suppress
   before touching this shard.

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
