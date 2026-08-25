---
title: "tests_without_assertions counter only sees vitest expect(), not the ported Minitest assert helpers"
status: draft
updated: 2026-08-16
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`active_support/testing/tests_without_assertions.rb:10-17` warns when a test
finished with `assertions == 0`. In Minitest that counter is incremented by
`Minitest::Assertions#assert` itself, so every `assert_*` helper — `assert_equal`,
`assert_predicate`, `assert_empty`, … — feeds it.

Our port reads the counter off vitest instead:
`packages/activesupport/src/test-case.ts:172` — `assertions: expect.getState().assertionCalls ?? 0`.
That counts only `expect()` calls, so a test built entirely out of the ported
Minitest helpers in `packages/activesupport/src/testing/assertions.ts`
(`assert`, `assertPredicate`, `assertEmpty`, `assertSame`, …) reports zero and
gets falsely warned about.

Observed on PR #6591: seven of the fourteen tests in
`packages/activemodel/src/validations/conditional-validation.test.ts` print
"Test is missing assertions" on every run despite each making two real
assertions. RFC 0105 pushes ports toward these helpers (they are what
`scripts/test-compare/assertion-kinds.ts` maps onto `predicate`/`empty`), so the
false-warning volume grows with every converged file.

## Converged shape

Have `assert` (`assertions.ts:517`, the one function every other helper funnels
through, mirroring Minitest) increment a per-test counter, and have
`test-case.ts:172` report `expect.getState().assertionCalls + <that counter>`
rather than the vitest number alone. Reset it per test where the `RunningTest`
is built.

## Acceptance criteria

- A test whose only assertions are `assert*` helpers does not print
  "Test is missing assertions".
- A test that genuinely asserts nothing still warns
  (`packages/activesupport/src/testing/test-without-assertions.test.ts` stays green).
- Mixed `expect(...)` + `assert*` tests count both.
