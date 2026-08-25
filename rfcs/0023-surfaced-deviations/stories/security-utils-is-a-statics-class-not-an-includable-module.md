---
title: "SecurityUtils is a statics-only class, so include SecurityUtils callers name it instead"
status: draft
updated: 2026-08-06
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

`ActiveSupport::SecurityUtils`
(`vendor/rails/activesupport/lib/active_support/security_utils.rb:6`) is a module
with `module_function`, so a class can `include SecurityUtils` and call
`secure_compare` / `fixed_length_secure_compare` as instance methods. Two Rails
classes do: `SecureCompareRotator`
(`vendor/rails/activesupport/lib/active_support/secure_compare_rotator.rb:33`)
and `MessageVerifier`
(`vendor/rails/activesupport/lib/active_support/message_verifier.rb`).

trails ports it as a statics-only class
(`packages/activesupport/src/security-utils.ts`), so the callers name it:
`SecurityUtils.secureCompare(...)` in
`packages/activesupport/src/secure-compare-rotator.ts`. Surfaced by PR #6145,
which ported `SecureCompareRotator`; cited in that file's JSDoc.

The settled trails idiom for `include SomeModule` is `include()` / `Included<>`
from `@blazetrails/activesupport`, or `this`-typed functions assigned to the
class (CLAUDE.md, "Module mixins"). Neither was applied here.

## Converged shape

`SecurityUtils` becomes an includable module in the trails idiom, and the
classes Rails writes `include SecurityUtils` on call `this.secureCompare(...)` —
so the call sites read as the Ruby does. The `security-utils` subpath export
(the adapter-pattern reason it is not on the main index) is unaffected.

## Acceptance criteria

- [ ] `SecureCompareRotator` calls `secureCompare` the way Rails' included
      module does, not through a class name.
- [ ] Any other caller Rails reaches through `include` is converted with it.
- [ ] `pnpm parity:api` / `pnpm parity:api:extra` deltas non-negative;
      `secure-compare-rotator.test.ts` and `message-verifier` suites green.
