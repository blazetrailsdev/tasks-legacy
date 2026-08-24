---
title: "Converge core-ext/module.test.ts onto module_test.rb's real test bodies"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Converge `core-ext/module.test.ts` onto `module_test.rb`'s real test bodies

## Context

Surfaced landing PR #6863, which converged `Module#delegate` onto
`ActiveSupport::Delegation.generate`. `packages/activesupport/src/core-ext/module.test.ts`
declares `describe("ModuleTest")` and carries Rails' test NAMES from
`vendor/rails/activesupport/test/core_ext/module_test.rb`, so `parity:test`
credits it — but a majority of the bodies are trails-invented approximations
that do not exercise what the Rails test exercises. Several are pure tautologies:

- `delegation to class method` (`module_test.rb:196-199` delegates to `:class`)
  asserts only `expect(typeof delegate).toBe("function")`;
- `delegate missing to with reserved methods` and `delegate missing to does not
delegate to private methods` are the same tautology;
- `delegation line number`, `module nesting is empty`, and
  `delegate missing to respects superclass missing` assert something unrelated
  to the Rails test of that name;
- most `delegate missing to *` tests call `delegate`, not `delegateMissingTo`,
  because until #6863 `delegateMissingTo` was a marker that did nothing.

PR #6998 converged one more, for the same reason #6863 converged its sample:
porting `Delegation.generate_method_missing`'s reserved / `__target` receiver
prefix (`delegation.rb:162`) made Rails' assertion implementable, so
`delegate missing to with reserved methods` now mirrors `module_test.rb:428`'s
`DecoratedReserved` (`:117-125`) — `delegate_missing_to :case`, `attr_reader
:case`, `.name` forwarding to the David person — via `delegateMissingTo` rather
than `delegate`. Strike it from the tautology list above; the rest stand.

PR #6863 converged one of them as a sample —
`delegation doesnt mask nested no method error on nil receiver` now mirrors
Rails' `Product` / `manufacturer` / `type` fixture (`module_test.rb:65-76,
405-413`) — and left the rest, since rewriting ~20 bodies is its own PR.

This matters beyond tidiness: a green tautology under a Rails test name is
counted as ported by `parity:test`, so the percentage overstates coverage for
exactly the members that have no real coverage.

## Converged shape

Each `it` body ports the Ruby method of the same name from `module_test.rb`,
against the fixtures the Ruby uses (`Someone`, `Somewhere`, `Invoice`,
`Project`, `DecoratedTester`, `Product`, `Tester`, `ParameterSet`, …, declared
at `module_test.rb:1-160`). Test NAMES do not change — they already match.
`delegate_missing_to` tests call `delegateMissingTo`, which is a real
implementation since #6863.

Where a Ruby body genuinely cannot port (Ruby method visibility, `__LINE__`
backtrace assertions), reclassify it in
`scripts/api-compare/unported-files.ts` rather than leaving a passing
tautology under the name.

## Acceptance criteria

- [ ] Every `it` in `core-ext/module.test.ts` either ports its `module_test.rb`
      body or is reclassified as unported; no body whose only assertion is
      about `delegate` being a function remains.
- [ ] The `delegate_missing_to` tests exercise `delegateMissingTo`.
- [ ] No test name changes.
- [ ] `pnpm parity:test` / `pnpm parity:test:assertions` deltas non-negative.
