---
title: "Fold predications-range.test.ts into predications.test.ts (its source file is gone)"
status: ready
updated: 2026-08-23
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 240
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6856 deleted `packages/arel/src/predications-range.ts` (folded into
`predications.ts`), but left `packages/arel/src/predications-range.test.ts` in
place: 37 behavior-level tests driving the public `between` / `notBetween`
surface, mirroring Rails' `activerecord/test/cases/arel/attributes/attribute_test.rb`
`#between` / `#not_between` blocks.

The file now names a source file that does not exist, which breaks the repo's
"tests live next to source files as `*.test.ts`" convention. It was not renamed
in #6856 because a test-file rename touches `parity:test` enrollment (the
test-compare manifest keys off file paths) and that was out of scope.

## Converged shape

Fold the cases into `packages/arel/src/predications.test.ts` (the Rails-mirroring
file for `predications.rb`), keeping every test NAME verbatim — they are matched
by `parity:test` and must not be reworded. Verify the enrollment registrations
before and after (see `scripts/test-compare/`), and confirm
`pnpm parity:test` deltas are non-negative.

## Acceptance criteria

- `packages/arel/src/predications-range.test.ts` is gone; every one of its 37
  cases lives in `predications.test.ts` under its existing name.
- `pnpm parity:test` non-negative; no test-compare manifest row lost.
- `pnpm vitest run packages/arel` green.
