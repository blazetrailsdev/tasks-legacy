---
title: "Pair assertion value tokens positionally instead of greedy strict-first matching"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6992 (`map-minitest-spec-assertion-forms`, RFC 0122).

`assertionValueMismatch` compares each canonical kind's literal tokens as a
multiset. PR #6992 introduced `LOOSE_RAILS_KINDS`
(`scripts/test-compare/assertion-values.ts`) so that an operand asserted through
a whitespace-squeezing Rails helper compares whitespace-insensitively while a
sibling `must_equal` in the same test still compares verbatim — Rails squeezes
only the target and `other` of that one helper call before delegating
(`vendor/rails/activerecord/test/cases/arel/helper.rb:10-13`).

The Rails side carries that attribution per assertion index. **The trails side
does not**: vitest spells both a `must_be_like` port and a `must_equal` port as
`toEqual`, so `collectSide` cannot know which trails token belongs to which
Rails assertion. `looseMismatch` therefore matches the strict tokens first,
greedily, and squeezes whatever is left over.

Where a strict token could equally have partnered a loose one, the greedy pass
consumes it and the result is a **false MISMATCH, never a false match** — the
safe direction, and the same rule `TRAILS_MAP` already applies to `toBe`. It is
documented at the call site, but it is still an imprecision, not a correct
comparison.

Measured today: 10 of the 326 arel tests using `must_be_like` mix it with
another `equal`-kind assertion, and none of the 10 currently mis-pairs — arel's
value counter is 79 either way. The exposure grows as more helpers gain loose
treatment, which is exactly what
`0023-surfaced-deviations/map-rails-helper-twin-assertion-kinds` proposes.

## Acceptance criteria

- The extractors (`scripts/test-compare/extract-ruby-tests.rb` and
  `extract-ts-core.ts`) emit enough per-assertion information — assertion order
  is already lockstep with `assertionKinds`/`assertionValues` — for
  `assertionValueMismatch` to pair tokens positionally rather than by greedy
  multiset consumption, at least when both sides have equal counts for the kind.
- `looseMismatch`'s greedy strict-first pass is replaced by that pairing, or is
  kept only as the documented fallback for the genuinely unpairable case.
- A regression test covers the mis-pairing shape the greedy pass gets wrong: a
  test whose `must_be_like` operand happens to equal the sibling `must_equal`
  operand verbatim, where positional pairing passes and greedy matching reports
  a false mismatch.
- No package's counters in `scripts/test-compare/assertion-mismatch-mark.json`
  rise. Report the before/after for all five gated packages; the mark is
  only-shrink.
