---
title: "wave-5g-tail-sweep"
status: ready
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
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

Measured after wave 5f, for `activerecord` / `arel` / `activesupport`:

- `activerecord/attribute-methods/primary-key.json` — 2 rows (ruled left whole
  in wave 5e: both rows say in prose that the async/sync schema-cache split is
  convergeable work the row tracks)
- `activerecord/delegated-type.json` — 3 rows. All three are entangled with the
  `delegated_type -> define_delegated_type_methods` decomposition divergence,
  which the row itself says is "a real decomposition divergence, not a language
  shortcoming; converging means moving the method-definition half into
  `defineDelegatedTypeMethods`. Tracked." The other two say "Converges with that
  row". CONVERGEABLE — this shard wants the decomposition fix, not receipts.
- `activerecord/querying.json` — 2 rows, both "Converges once `_loadFromSql`
  takes the Result". CONVERGEABLE.

Regenerate the list with

    pnpm build && API_COMPARE_FORCE=1 pnpm parity:api --calls

then group `call-mismatches-exclude/**` by file at `kind: "set"`, keeping the
files with 2 or 3 rows for those three packages.

Every one of the three remaining shards is CONVERGEABLE by the RFC 0106 rule at
`missing-rails-call-tags.ts:296-299` — a deviation whose work is still owned
belongs in `call-mismatches-exclude/` as a row. So this story is NOT another
receipt-migration sweep: it converges the ports, or it blocks on the named
owners.

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
