---
title: "wave-5f-tail-sweep"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6898
claim: "2026-08-23T00:12:27Z"
assignee: "wave-5f-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Continuation of `wave-5e-tail-sweep` (RFC 0106), which took 14 of the 28
`kind: "set"` tail shards listed there — 33 rows migrated to
`@missingRailsCall` receipts, 11 shards deleted, three shards left holding
their remaining rows (`postgresql/quoting.json` `quote_string`,
`mysql2/database-statements.json`'s `args` row,
`encryption/cipher/aes256-gcm.json`'s `args` row).

What remains of the 2/3-row `kind: "set"` tail for `activerecord` / `arel` /
`activesupport`, measured after that PR: **22 files, 55 rows**.

- `activerecord/associations/alias-tracker.json` — 2 rows
- `activerecord/associations/association-scope.json` — 2 rows
- `activerecord/associations/builder/belongs-to.json` — 2 rows
- `activerecord/associations/collection-association.json` — 3 rows
- `activerecord/associations/has-many-association.json` — 2 rows
- `activerecord/associations/join-dependency/join-association.json` — 2 rows
- `activerecord/associations/join-dependency.json` — 3 rows
- `activerecord/associations/preloader/through-association.json` — 3 rows
- `activerecord/attribute-methods.json` — 3 rows
- `activerecord/attribute-methods/primary-key.json` — 2 rows
- `activerecord/core.json` — 3 rows
- `activerecord/delegated-type.json` — 3 rows
- `activerecord/middleware/database-selector/resolver.json` — 3 rows
- `activerecord/migration/command-recorder.json` — 3 rows
- `activerecord/querying.json` — 2 rows
- `activerecord/relation/batches.json` — 3 rows
- `activerecord/relation/where-clause.json` — 3 rows
- `activerecord/scoping/default.json` — 2 rows
- `activerecord/secure-token.json` — 2 rows
- `activerecord/statement-cache.json` — 3 rows
- `activerecord/transactions.json` — 2 rows
- `activesupport/inflector.json` — 2 rows

Regenerate the list with

    pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls

then group `call-mismatches-exclude/**` by file at `kind: "set"`, keeping the
files with 2 or 3 rows for those three packages.

This is likely more than one 700 LOC PR. Ship it as sequential
non-overlapping PRs from `main`, never stacked, and file the remainder again
as a follow-on.

### Rulings carried forward from waves 5c/5d/5e

1. **A migrated tag needs a permanence claim.** `parity:api:build` writes the
   baseline `reason` verbatim, and `suppressedCallsIn`
   (`scripts/api-compare/missing-rails-call-tags.ts:219-230`) hard-errors
   unless the reason opens with `PERMANENT` or `CONVERGEABLE`. Prefix each
   migrated reason with `PERMANENT:` in the baseline JSON first (through
   `serializeBaseline`, never a hand-edit), then run
   `pnpm parity:api:build --package <pkg> --file <tsFile> [--call <ruby_call>]`.
   Note `--call` takes the bare Ruby call name, not `rubyName:call`.

2. **A CONVERGEABLE row must NOT migrate.** Per the RFC 0106 rule at
   `missing-rails-call-tags.ts:296-299`, a deviation whose work is still owned
   belongs in `call-mismatches-exclude/` as a row. Rows left baselined on this
   basis so far: `collection-association.json` `transaction`,
   `join-dependency.json` `match?`, `preloader/through-association.json`
   `scope` (wave 5d), and `postgresql/quoting.json` `quote_string` (the RFC
   0073 permanent-checkout flip, wave 5e). `attribute-methods/primary-key.json`
   was inspected in wave 5e and left whole: both its rows say in prose that the
   async/sync schema-cache split is convergeable work the row tracks.

3. **`activesupport/inflector.json` is a known trap — handle it separately.**
   Its two rows (`safe_constantize` -> `const_regexp` / `match?`) coexist on
   `main` with `@missingRailsCall` tags for the SAME calls on
   `safeConstantize`'s JSDoc. Dropping the rows reports both tags STALE;
   dropping the tags as well then reports both calls as NEW mismatches. The tag
   is not suppressing the flag, so the row and the tag are not interchangeable
   here. Waves 5c, 5d and 5e all left it alone. Diagnose why the tag on an
   exported top-level `function` in `inflector.ts` fails to suppress before
   touching this shard.

## Acceptance criteria

- [ ] Every row in the files taken by the PR is converged, or leaves as a
      `@missingRailsCall` tag carrying its reviewed per-site reason, prefixed
      with a `PERMANENT` claim, at the call site. Never a name-keyed bulk edit,
      a broadened reason, or a move to another register.
- [ ] Rows whose reason names a convergence OWNER stay baseline rows.
- [ ] Emptied shards deleted, not committed as `[]`;
      `pnpm parity:api:calls:tighten <shard>` for any shard left with a stale
      mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
- [ ] The un-taken remainder is filed as a follow-on story with the current
      file list.
