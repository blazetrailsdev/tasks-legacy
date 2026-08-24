---
title: "Merge the colliding postgresql/schema-statements-class test-file pair"
status: claimed
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 450
priority: null
pr: null
claim: "2026-08-24T02:27:39Z"
assignee: "merge-colliding-test-file-pair-mysql-schema-statements"
blocked-by: null
closed-reason: null
---

## Context

Fourth and last of the collision pairs left behind by
`sweep-trails-only-test-files-connection-adapters` (#6950); its three siblings
are already filed as `merge-colliding-test-file-pair-abstract-schema-definitions`,
`-mysql-schema-creation`, and `-mysql-schema-statements`.

Two files describe the same trails-only unit under
`packages/activerecord/src/connection-adapters/postgresql/`:

- `schema-statements-class.test.ts` (212 L) — `SchemaStatements#dropTable`,
  `#dropSchema`, `#schemaSearchPath`, `#addForeignKey use_foreign_keys? guard`
- `schema-statements-class.trails.test.ts` (392 L) — constraint name digests,
  sequence helpers, `#indexNameExists`, `#pkAndSequenceFor`,
  `#resetPkSequenceBang`, `sequenceNameFromParts`, `#typeToSql` enum
  validation, `#changeTable`

The plain-named one could not be renamed onto `.trails.` in #6950 because the
`.trails.test.ts` path was already taken, and merging it there would have cost
the whole file in both insertions and deletions (git cannot pair a delete of
`X.test.ts` with a modify of a pre-existing `X.trails.test.ts`), blowing the
PR's LOC ceiling.

Both files are trails inventions, not names a future Rails port would claim:
`rubyToConventionTs` (`scripts/test-compare/compare.ts:135-173`) maps every
Rails `*_test.rb` deterministically, and only
`test/cases/connection_adapters/*_test.rb` — 13 files, all already ported —
lands under `connection-adapters/`. The nearest-named Rails file,
`activerecord/test/cases/adapters/postgresql/schema_test.rb:587`, maps to
`adapters/postgresql/schema.test.ts`, which is already ported.

## Acceptance criteria

- [ ] Merge the two into a single
      `connection-adapters/postgresql/schema-statements-class.trails.test.ts`,
      preserving every `describe`/`it` name verbatim — no test-name edits.
- [ ] Update `eslint/require-canonical-rebuild-exclude.json`, which names
      `connection-adapters/postgresql/schema-statements-class.test.ts` (the
      `nonExecuting` list); keep the list sorted.
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
