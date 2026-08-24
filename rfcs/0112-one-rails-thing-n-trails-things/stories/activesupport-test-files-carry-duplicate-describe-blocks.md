---
title: "key-generator.test.ts and parameter-filter.test.ts register duplicate describe blocks and a misplaced BacktraceCleaner copy"
status: claimed
updated: 2026-08-24
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 130
pr: null
claim: "2026-08-24T02:13:27Z"
assignee: "activesupport-delegation-module-port"
blocked-by: null
closed-reason: null
---

## Context

Two activesupport test files contain duplicated `describe` blocks that register
the same test names twice, which makes `parity:test` measure whichever copy the
extractor reaches and hides the other:

- `packages/activesupport/src/key-generator.test.ts` — `describe("KeyGeneratorTest")`
  and `describe("CachingKeyGeneratorTest")` each appear **twice**, the second
  `KeyGeneratorTest` being six bare `it.skip(...)` calls with no body. The file
  also carries a verbatim copy of five BacktraceCleaner suites
  (`BacktraceCleanerFilterTest`, `BacktraceCleanerSilencerTest`,
  `BacktraceCleanerMultipleSilencersTest`,
  `BacktraceCleanerFilterAndSilencerTest`, plus `#dup` cases) that already live
  in their correct home, `packages/activesupport/src/backtrace-cleaner.test.ts`
  — mapped to `vendor/rails/activesupport/test/backtrace_cleaner_test.rb`, not
  to `key_generator_test.rb`.
- `packages/activesupport/src/parameter-filter.test.ts` — `describe("ParameterFilterTest")`
  appears twice, the second copy re-declaring all 8 Rails test names plus 6
  trails-only extras.

Found while converging assertion parity in PR #6640; deliberately left alone
there to keep that diff scoped to assertion changes.

## Converged shape

One `describe` per Rails test class, in the file whose convention path maps to
that class's `.rb`. Delete the duplicate blocks and the misplaced
BacktraceCleaner copies (the originals in `backtrace-cleaner.test.ts` stay);
move any trails-only extras that are worth keeping into the sibling
`*.trails.test.ts` file per CLAUDE.md, and delete the empty `it.skip` stubs
outright — an unported test already has its PERMANENT-SKIP stub elsewhere, and a
bodyless `it.skip` inside a live describe is not one.

## Acceptance criteria

- No test name is registered twice in either file.
- `pnpm parity:test --package activesupport` percent does not drop and
  `0 misplaced` holds.
- No new rows in `scripts/parity/unported-files/`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
