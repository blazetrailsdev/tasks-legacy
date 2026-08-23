---
title: "merge-colliding-trails-test-file-pairs-connection-adapters"
status: claimed
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T21:44:31Z"
assignee: "merge-colliding-trails-test-file-pairs-connection-adapters"
blocked-by: null
closed-reason: null
---

## Context

`sweep-trails-only-test-files-connection-adapters` renamed 64 of the 68
trails-invention test files under
`packages/activerecord/src/connection-adapters/` onto `*.trails.test.ts`.
Four could not be renamed: a `*.trails.test.ts` sibling already exists at the
target path, so the rename collides.

| plain (trails invention, still unrenamed)                                | existing sibling          |
| ------------------------------------------------------------------------ | ------------------------- |
| `connection-adapters/abstract/schema-definitions.test.ts` (519 L)        | `.trails.test.ts` (299 L) |
| `connection-adapters/mysql/schema-creation.test.ts` (361 L)              | `.trails.test.ts` (31 L)  |
| `connection-adapters/mysql/schema-statements.test.ts` (689 L)            | `.trails.test.ts` (17 L)  |
| `connection-adapters/postgresql/schema-statements-class.test.ts` (212 L) | `.trails.test.ts` (392 L) |

All four plain files are trails inventions by the same test:
`rubyToConventionTs` (`scripts/test-compare/compare.ts:135`) maps every Rails
`*_test.rb` under `test/cases/` deterministically, and only
`cases/connection_adapters/*_test.rb` (13 files, all already ported) lands
under `connection-adapters/`. A future port of, say,
`cases/adapters/postgresql/quoting_test.rb` goes to
`adapters/postgresql/quoting.test.ts`, never `connection-adapters/…`.

The fix is to merge each pair into one `*.trails.test.ts` file. It was left
out of the sweep PR purely on size: git cannot pair a delete of
`X.test.ts` with a modify of the pre-existing `X.trails.test.ts`, so each
merge costs the full file in both insertions and deletions — roughly 2100 LOC
for the mysql pair alone and ~3600 for all four, well past the PR ceiling.

## Acceptance criteria

- [ ] Merge each of the four pairs into a single `*.trails.test.ts` file,
      preserving every `describe`/`it` name verbatim (no test-name edits).
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
- [ ] Update the paths that name the plain files:
      `eslint/no-explicit-any-test-exclude.json` (`abstract/schema-definitions.test.ts`,
      `mysql/schema-creation.test.ts`) and
      `eslint/require-canonical-rebuild-exclude.json`
      (`postgresql/schema-statements-class.test.ts`).
- [ ] Split across as many PRs as the LOC ceiling requires — one pair per PR
      is fine.
