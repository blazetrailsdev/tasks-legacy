---
title: "merge-colliding-test-file-pair-mysql-schema-statements"
status: done
updated: 2026-08-24
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6967
claim: "2026-08-24T02:27:39Z"
assignee: "merge-colliding-test-file-pair-mysql-schema-statements"
blocked-by: null
closed-reason: null
---

## Context

Follow-up to `merge-colliding-trails-test-file-pairs-connection-adapters`
(RFC 0078), which merged only the postgresql pair — the merges are
size-bound, and the other three did not fit the 700 LOC ceiling in one PR.

Remaining colliding pair under
`packages/activerecord/src/connection-adapters/`:

| plain (trails invention, still unrenamed) | existing sibling                                |
| ----------------------------------------- | ----------------------------------------------- |
| `mysql/schema-statements.test.ts` (689 L) | `mysql/schema-statements.trails.test.ts` (17 L) |

Both files are trails inventions: `rubyToConventionTs`
(`scripts/test-compare/compare.ts:135`) only maps
`cases/connection_adapters/*_test.rb` (13 files, all ported) under
`connection-adapters/`, so no future Rails port ever lands at either path.

The merge pattern is the one PR #<pg> used: fold the plain file's helpers and
`describe` blocks into the `.trails.test.ts` sibling, de-duplicating any
helper both copies define, merge the import lists, and delete the plain file.

## Acceptance criteria

- [ ] `mysql/schema-statements.test.ts` is merged into `mysql/schema-statements.trails.test.ts` and deleted,
      preserving every `describe`/`it` name verbatim (no test-name edits).
- [ ] `pnpm parity:test --package activerecord` totals byte-identical before
      and after.
- [ ] Any eslint exclude path naming the plain file is updated (none).
- [ ] The merged file passes `pnpm vitest run <path>`.
