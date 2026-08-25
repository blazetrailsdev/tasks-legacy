---
title: "Fold bespoke security.test.ts message coverage into the Rails-named suites"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/security.test.ts` is a trails-invented test file
with no Rails counterpart. It re-tests `MessageEncryptor` and `MessageVerifier`
behaviour that Rails already covers in
`vendor/rails/activesupport/test/message_encryptor_test.rb` and
`message_verifier_test.rb`:

- `security.test.ts:32-37` `raises on tampered encrypted value` duplicates
  `message_encryptor_test.rb:44-51`
  (`test_messing_with_verified_values_causes_failures`).
- `security.test.ts:93-97` `valid_message returns false on tampered data`
  duplicates `message_verifier_test.rb:29-37` (`test_valid_message`).

Because the file's test names have no Rails counterpart, `parity:test` counts
them as "extra (TS only)" rather than as coverage. The duplicated assertions
belong in the Rails-named suites where they can be matched.

Note the tamper mechanics here are sound — `split("").reverse().join("")` is a
whole-string reverse, not the 1-in-16-collision-prone single-character poke that
PR #5388 fixed in `codec.trails.test.ts`. This is a layout/fidelity issue, not a
flake.

## Acceptance criteria

- Coverage unique to `security.test.ts` moves into the Rails-named suites
  (`message-encryptor.test.ts`, `message-verifier.test.ts`) under the Rails test
  names, or into a `*.trails.test.ts` file if genuinely TS-only.
- Assertions that merely duplicate existing Rails-named tests are dropped rather
  than copied.
- `parity:test` "extra (TS only)" count for activesupport does not increase.
- `pnpm vitest run packages/activesupport/src/`, `pnpm typecheck`, `pnpm lint`
  pass.
