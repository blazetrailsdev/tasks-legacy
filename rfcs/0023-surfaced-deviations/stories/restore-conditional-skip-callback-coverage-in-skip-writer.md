---
title: "skip writer test is weakened to an unconditional skip"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `WriterCallbacksTest#test_skip_writer`
(activesupport/test/callbacks_test.rb:908-935) exercises `skip_callback` with a
run-time conditional: `WriterSkipper < Person` declares
`skip_callback :save, :before, :before_save_method, if: -> { age > 21 }`
(:910), and the test asserts that an 18-year-old writer still runs the whole
inherited `before_save` chain — the skip does NOT apply, because the condition
is false. `skip_callback` with `:if` rewrites the chain entry via
`merge_conditional_options` (callbacks.rb:800-804) instead of deleting it.

trails' port (`packages/activesupport/src/callbacks.ts` →
`packages/activesupport/src/callbacks.test.ts:1232-1241`, `describe
"WriterCallbacksTest" > it "skip writer"`) keeps the Rails test name but has
been weakened to an unconditional skip on a two-entry chain: it registers one
callback, skips it with no `:if`, and asserts it did not run. Neither the
conditional nor the inheritance the Rails test is about is covered.

The engine half is already in place —
`Callback#mergeConditionalOptions` and the `"if" in options || "unless" in
options` arm of `Callbacks.skipCallback` — and PR #6951 added a Model-level
conditional-skip test at `packages/activemodel/src/callbacks.test.ts`. So this
is a test-fidelity gap, not a behaviour gap.

## Converged shape

Port `test_skip_writer` as Rails writes it: a `Person`-shaped base with the
five `before_save` filter kinds (symbol, proc, object, class, block) already
present in that file's `Person` port, a `WriterSkipper` subclass whose
`skipCallback(…, { if: … })` reads `age`, and the two assertions — the full
history at age 18, and the skipped entry missing at age 22 (Rails covers the
true arm through `WriterSkipper` in the surrounding suite). Keep the `it("skip
writer")` name exactly; only the body changes.

Related: `test_skip_undefined_callback` (:1175) and `test_skip_without_raise`
(:1184) are already faithfully ported at callbacks.test.ts:1798-1816 — use them
as the shape reference.

## Acceptance criteria

- [ ] `it("skip writer")` mirrors callbacks_test.rb:908-935: inherited chain,
      conditional `if:`, both arms of the condition.
- [ ] The test fails on the baseline if the conditional arm of
      `skipCallback` is removed.
- [ ] Test name unchanged; no new bespoke helper classes beyond the Rails ones.
- [ ] `pnpm parity:test` delta non-negative.
