---
title: "wave-5g-tail-sweep"
status: done
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6901
claim: "2026-08-23T01:12:27Z"
assignee: "wave-5g-tail-sweep"
blocked-by: null
closed-reason: null
---

## Context

Continuation of `wave-5f-tail-sweep` (RFC 0106), which took 19 of the 22
`kind: "set"` tail shards listed there — 39 rows migrated to `@missingRailsCall`
receipts, 12 shards deleted, 7 shards left holding their remaining rows.

Wave 5f also fixed the `activesupport/inflector.json` trap wave 5c/5d/5e all
left alone: `staleCallTags` (`scripts/api-compare/compare.ts`) keyed a
TOP-LEVEL function's staleness on the `""`-owner `used` set alone, while the
pair that actually raised (and suppressed) the flags resolved its owner to
`Inflector`. One exported `function safeConstantize` is the TS side of two
pairs; only the owned one flags. The artifact therefore listed the same call as
BOTH `suppressed` and `staleTags`. Fixed by `usedForAnyOwner()`: a `""`-owner
tag is credited with every owner's suppressions for its (tsFile, tsName); a
real owner is still matched exactly, so a sibling class's tag never borrows
another's suppression.

### What remains of the 2/3-row `kind: "set"` tail

Measured after wave 5f, for `activerecord` / `arel` / `activesupport`: **18
files**. Wave 5f's own story enumerated only the 22 shards it had measured at
the time; the full 2/3-row `kind: "set"` band for those packages is wider than
that enumeration, and the shards below marked NEW were never listed by wave 5e
or 5f. Regenerate before claiming:

    pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls

then group `call-mismatches-exclude/**` by file at `kind: "set"`, keeping the
files with 2 or 3 rows for those three packages.

#### Ruled CONVERGEABLE — these want the port fixed, not a receipt

- `activerecord/attribute-methods/primary-key.json` — 2 rows (ruled left whole
  in wave 5e: both rows say in prose that the async/sync schema-cache split is
  convergeable work the row tracks)
- `activerecord/delegated-type.json` — 3 rows. All three are entangled with the
  `delegated_type -> define_delegated_type_methods` decomposition divergence,
  which the row itself says is "a real decomposition divergence, not a language
  shortcoming; converging means moving the method-definition half into
  `defineDelegatedTypeMethods`. Tracked." The other two say "Converges with that
  row".
- `activerecord/querying.json` — 2 rows, both "Converges once `_loadFromSql`
  takes the Result".

By the RFC 0106 rule at `missing-rails-call-tags.ts:296-299` — a deviation whose
work is still owned belongs in `call-mismatches-exclude/` as a row — none of
these three may migrate to a `@missingRailsCall` receipt.

#### Known residue from wave 5e, left holding their remaining rows

- `activerecord/connection-adapters/postgresql/quoting.json` — 3 rows
  (`quote_string` is the RFC 0073 permanent-checkout flip, CONVERGEABLE)
- `activerecord/connection-adapters/mysql2/database-statements.json` — 2 rows
- `activerecord/encryption/cipher/aes256-gcm.json` — 2 rows
- `activerecord/statement-cache.json` — 2 rows (wave 5f took `create -> call`;
  the two left are `execute -> async_find_by_sql` / `-> wrap`, both tracked by
  story `port-promise-complete-for-async-loaded-arms`, RFC 0023)

#### NEW — never enumerated by wave 5e or 5f, unclassified

Each needs the same per-site pass wave 5f ran: read the Rails body, decide
PERMANENT (migrate to a receipt with a `PERMANENT:` claim) or CONVERGEABLE
(leave the row).

- `activerecord/connection-adapters/abstract/schema-creation.json` — 3 rows
- `activerecord/connection-adapters/mysql/column.json` — 2 rows
- `activerecord/connection-adapters/pool-config.json` — 2 rows
- `activerecord/connection-adapters/postgresql/schema-dumper.json` — 2 rows
- `activerecord/connection-adapters/sqlite3/database-statements.json` — 2 rows
- `activerecord/connection-adapters/sqlite3/schema-statements.json` — 3 rows
- `activerecord/connection-handling.json` — 2 rows
- `activerecord/disable-joins-association-relation.json` — 2 rows
- `activerecord/encryption/encryptable-record.json` — 2 rows
- `activerecord/encryption/extended-deterministic-queries.json` — 3 rows
- `activerecord/validations/associated.json` — 3 rows

This is likely more than one 700 LOC PR. Ship it as sequential non-overlapping
PRs from `main`, never stacked, and file the remainder again as a follow-on.

### Rulings carried forward

1. A migrated tag needs a permanence claim: prefix the baseline `reason` with
   `PERMANENT:` (through `serializeBaseline`, never a hand-edit), then
   `pnpm parity:api:build --package <pkg> --file <tsFile> [--call <ruby_call>]`.
   `--call` takes the bare Ruby call name.
2. `--call <name>` migrates EVERY row with that call name on the file, not just
   the one you meant. Wave 5f hit this on `scoping/default.json`, where a
   CONVERGEABLE `build_default_scope -> any?` row shared the call name with the
   PERMANENT `scope_attributes? -> any?` row and was migrated without a
   permanence claim; the fix was to restore the row and delete the tag by hand.
   Check the migrated count against what you intended, every time.
3. A CONVERGEABLE row must NOT migrate.

## Acceptance criteria

- [ ] `activerecord/querying.json` — `_loadFromSql` takes the `Result`, so
      `includes_column?(inheritance_column)` and the `instantiate_instance_of`
      arm are both made; the two rows converge by deletion. Or the story blocks
      with the specific blocker.
- [ ] `activerecord/delegated-type.json` — the method-definition half moves
      into `defineDelegatedTypeMethods`, so `delegatedType` calls it in the
      Rails direction (`delegated_type.rb:233`); all three rows converge by
      deletion. Or the story blocks.
- [ ] `activerecord/attribute-methods/primary-key.json` — converge the
      async/sync schema-cache split, or re-affirm the wave 5e ruling and leave
      it, saying why in the PR body.
- [ ] Emptied shards deleted, not committed as `[]`;
      `pnpm parity:api:calls:tighten <shard>` for any shard left with a stale
      mark.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
