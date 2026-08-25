---
title: "BigDecimal's constructor raises TypeError where Kernel#BigDecimal raises ArgumentError"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing PR #6790 (move-decimal-rounding-into-activesupport-bigdecimal).

Ruby raises `ArgumentError` from `Kernel#BigDecimal` for both failure modes
trails' `BigDecimal` constructor has:

    BigDecimal("nonsense")      # => ArgumentError: invalid value for BigDecimal(): "nonsense"
    BigDecimal(Rational(1, 3))  # => ArgumentError: can't omit precision for a Rational.

(Both verified on MRI 3.3.) `packages/activesupport/src/core-ext/big-decimal/conversions.ts`
throws `TypeError` at both sites instead:

- the constructor's `throw new TypeError(\`BigDecimal: cannot parse ${...}\`)`
  (pre-existing), and
- `parseRational`'s `throw new TypeError("can't omit precision for a Rational.")`
  (added by #6790, with the Ruby message but the wrong class).

The reason is structural, and it is cited at the second call site: this module
has no runtime imports at all. ActiveSupport's `ArgumentError` lives in
`packages/activesupport/src/hash-utils.ts`, which imports
`core-ext/object/blank.js`, which imports `@blazetrails/date`'s `Temporal` and
`time-with-zone.js` — so importing it would hang that whole graph off a file
that today joins no cycle and is imported by the numeric primitives. That is a
real constraint but not a language shortcoming, so it is debt, not a
ratifiable deviation.

CLAUDE.md's "Errors" rule is same error class, same message string, same raise
site. The message is already Ruby's at both sites; only the class is wrong.

## Converged shape

Give activesupport an import-free error module the numeric primitives can
reach — the same shape as the sanctioned zero-import slot modules
(`configurable-slot.ts`, `collection-proxy-slot.ts`), except it exports a
class rather than a mutable binding:

- Move `ArgumentError` out of `hash-utils.ts` into its own module with no
  runtime imports, re-exported from `hash-utils.ts` so its existing importers
  are unchanged and `packages/activesupport/src/index.ts` keeps one export.
- `conversions.ts` imports it and throws `ArgumentError` at both sites, with
  Ruby's messages: `invalid value for BigDecimal(): "<value>"` for the parse
  failure (currently a trails-invented `BigDecimal: cannot parse <value>`
  string) and the existing `can't omit precision for a Rational.`.
- Delete the "this module has no runtime imports by construction" paragraph in
  `parseRational`, which exists only to justify the wrong class.

Note `two-argumenterror-classes-break-instanceof` (0023, closed) and
`assert-valid-keys-argumenterror-type` (0023, done) for the prior art on which
`ArgumentError` is canonical — `cache/store.ts` and
`messages/serializer-with-fallback.ts` each still declare their own.

## Acceptance criteria

- [ ] `packages/activesupport/src/core-ext/big-decimal/conversions.ts` throws
      `ArgumentError`, not `TypeError`, for an unparseable value and for a
      Rational with no precision.
- [ ] The unparseable-value message is Ruby's
      `invalid value for BigDecimal(): "<value>"`.
- [ ] The module `conversions.ts` imports `ArgumentError` from has no runtime
      imports of its own, so `conversions.ts` still joins no import cycle —
      verified with a plain-node import of the built `dist/**.js` module as
      the entry module, not through vitest.
- [ ] The JSDoc paragraph in `parseRational` justifying `TypeError` is
      deleted, not reworded.
- [ ] `packages/activesupport/src/core-ext/bigdecimal.test.ts` asserts the
      class and message at both raise sites.
