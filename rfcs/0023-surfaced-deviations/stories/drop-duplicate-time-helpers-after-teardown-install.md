---
title: "Drop the duplicate TimeHelpers#after_teardown install in cases/helper.ts"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/cases/helper.ts:183-185` installs an `afterEach` that
calls `TimeHelpers#after_teardown` (`time_helpers.rb:70-73`) for the AR suite,
which was the right stand-in while `ActiveSupport::TestCase` had no receiver:

```ts
// Mirror `ActiveSupport::Testing::TimeHelpers#after_teardown` ...
afterEach(() => {
  afterTeardown();
});
```

PR #6516 gave the receiver the whole `after_teardown` chain — the `:teardown`
callbacks, `TimeHelpers#after_teardown`, then the missing-assertions check —
installed once in `packages/activesupport/src/test-case.ts`, which is a setup
file of every vitest project. Rails gets `after_teardown` exactly once, through
the `super` chain (`test_case.rb:144-151`); trails now runs the TimeHelpers arm
twice for AR tests. `travelBack` is idempotent, so this is redundancy rather
than a bug, but it is a second installation site Rails does not have and the
next reader has to work out which one wins.

## Converged shape

Delete the `afterEach` at `cases/helper.ts:183-185` and let the receiver's chain
be the single installation, as `test_case.rb` has it. Check
`packages/activerecord/src/cases/time-helpers-teardown.trails.test.ts` — it
exists to prove that wiring and should now prove the receiver's.

## Acceptance criteria

- `TimeHelpers#after_teardown` is installed at exactly one site.
- `time-helpers-teardown.trails.test.ts` still passes and points at the
  receiver.
