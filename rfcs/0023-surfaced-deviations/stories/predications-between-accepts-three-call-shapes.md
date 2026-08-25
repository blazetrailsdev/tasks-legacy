---
title: "between/notBetween take three call shapes where Rails takes one Range"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "arel"
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while folding `predications-range.ts` into `predications.ts` (#6856).

Rails' `between` / `not_between` (`activerecord/lib/arel/predications.rb:36`,
`:84`) take **one** argument — a Ruby `Range` — and read `other.begin`,
`other.end`, `other.exclude_end?` off it.

trails accepts three call shapes: `[begin, end]`, `{ begin, end, excludeEnd? }`,
and positional `(begin, end, excludeEnd?)`. That costs:

- a file-private `parseRange` normalizer in
  `packages/arel/src/predications.ts` (~:140, no Rails counterpart),
- a three-overload cast on both `between` and `notBetween`,
- the same overload set re-declared on `Attribute`
  (`packages/arel/src/attributes/attribute.ts:217-222`), because `Included<>`
  keeps only the last overload.

The arity divergence is also why `between` cannot be compared on argument shape
by `parity:api:calls:args`.

## Converged shape

One argument, one Range-like shape — pick the single spelling that is the JS
analogue of a Ruby Range (the `{ begin, end, excludeEnd? }` object is the closest
and is what AR's RangeHandler already threads) and drop the other two arms.
`parseRange`, both overload triples, and the `Attribute` re-declaration then
disappear.

Callers to migrate: grep `\.between(` / `\.notBetween(` across
`packages/arel/src` and `packages/activerecord/src` (plus the suites) — the
positional and array forms are used in tests more than in library code.

Related idiom class: RFC 0082 (`0082-ruby-ts-idiom-conversion-classes`), Ruby
Range protocol → JS.

## Acceptance criteria

- `between` / `notBetween` take one Range-like argument, matching
  `predications.rb:36` / `:84` arity.
- `parseRange` and both three-overload casts are gone from `predications.ts`;
  the `between` / `notBetween` re-declaration block in `attribute.ts` is gone
  or reduced to a single signature.
- `pnpm parity:api` arity for `Arel::Predications#between` / `#not_between`
  matches; no new `arity-exclude.json` row.
- `pnpm vitest run packages/arel` green, plus the AR range/where suites.
