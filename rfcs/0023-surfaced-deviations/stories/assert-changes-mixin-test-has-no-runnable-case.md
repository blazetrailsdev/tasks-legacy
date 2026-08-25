---
title: "assert_changes mix-in test cannot run a test case"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`test_assert_changes_when_assertions_are_included`
(`vendor/rails/activesupport/test/testing/method_call_assertions_test.rb:189-201`)
builds a real test case, RUNS it, and asserts the run passed:

```ruby
test_unit_class = Class.new(Minitest::Test) do
  include ActiveSupport::Testing::Assertions
  def test_assert_changes
    counter = 1
    assert_changes(-> { counter }) { counter = 2 }
  end
end

test_results = test_unit_class.new(:test_assert_changes).run
assert_predicate test_results, :passed?
```

The point of the test is that the assertions module works when MIXED INTO a
test case — not that `assert_changes` works standalone. PR #6650 ported it as a
`{ passed: false }` flag set after `assertChanges` returns
(`packages/activesupport/src/testing/method-call-assertions.test.ts`, the last
test), which exercises `assertChanges` but not the mix-in path, and the
deviation is cited at the call site.

The blocker is that trails has no runnable test-case class: vitest owns the run
loop and exposes no "instantiate this case, run it, hand me the result" entry
point. `Minitest::Runnable#run` returns `self` with `failures` populated, and
`passed?` reads `failures.empty?` (minitest 5.20.0, `lib/minitest.rb:337-360`,
`:451-469`).

Note trails already models the pieces: `testing/tests-without-assertions.ts`
defines a `RunningTest` interface standing in for the `self` of a
`Minitest::Test` (assertions / skipped / error / name / failures), built per
test from vitest's context by `test-case.ts`. A runnable analogue would reuse
that shape rather than invent a second one.

## Converged shape

Port enough of `Minitest::Runnable#run` / `#passed?` — over the existing
`RunningTest` shape — that the test can construct a case including the
assertions module, run it, and assert `passed?`, the way Rails does. If that
proves to require a test runner trails deliberately does not have, block the
story with that finding rather than re-justifying the flag.

## Acceptance criteria

- The test exercises the mix-in path (assertions module included into a case
  that is then run), not a bare `assertChanges` call.
- Assertion count / kind / value for that test stay at 0 mismatches in
  `pnpm parity:test -- --assertions --package activesupport`.
- The call-site deviation comment is deleted, not reworded.
