---
title: "Map the Rails-helper-twin assertion kinds both sides leave unmapped"
status: draft
updated: 2026-08-24
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

## Context

Surfaced while triaging RFC 0122 (arel assertion parity). It is cross-package,
so it is not arel's to own.

`parity:test`'s assertion comparison normalizes both sides onto the closed
`CanonicalKind` vocabulary in `scripts/test-compare/assertion-kinds.ts:20-44`.
Measured over **every** trails assertion token in
`scripts/test-compare/output/ts-tests.json` — 58,551 of them — **2,172 (3.7%)**
normalize to `null` and are reported as _unmapped_ rather than compared.

The tail is not vitest matchers. `TRAILS_MAP` already covers what our tests
actually use; the largest unmapped groups are trails helpers that mirror a
**Rails** helper whose Ruby twin is also absent from `RAILS_MAP`:

| trails token                                                      | count | Ruby twin, also unmapped                        |
| ----------------------------------------------------------------- | ----- | ----------------------------------------------- |
| `assertQueriesCount`                                              | 230   | `assert_queries_count`                          |
| `assertNoQueries`                                                 | 189   | `assert_no_queries`                             |
| `assertQueriesMatch`                                              | 102   | `assert_queries_match`                          |
| `assertInvalidValues` / `assertValidValues`                       | 135   | `assert_invalid_values` / `assert_valid_values` |
| `assertDifference` / `assertNoDifference`                         | 62    | `assert_difference` / `assert_no_difference`    |
| `assertChanges` / `assertClears`                                  | 32    | `assert_changes`                                |
| `assertParses`, `assertLookupType`, `assertCycle`, `assertXml`, … | ~300  | suite-local Rails helpers                       |

Because BOTH sides are unmapped, these pairs are neither credited as matching
nor flagged as divergent — they are simply invisible to the kind and value
dimensions. A test that ported `assert_queries_count(2)` as
`assertQueriesCount(5)` reads as clean today.

Genuinely vitest-native and unmapped is a much smaller set with no Minitest
counterpart by construction: the mock family (`toHaveBeenCalledWith` 179,
`toHaveBeenCalled` 72 + 74 negated, `toHaveBeenCalledTimes` 81,
`toHaveBeenCalledOnce` 28), plus `toHaveProperty` 112, `expectTypeOf` 31,
`toBeTypeOf` 26, `toMatchSnapshot` 26. Those should stay unmapped — forcing a
one-sided kind into a bucket is exactly the dishonesty
`assertion-kinds.ts:8-11` warns against.

Prior art for the shape of the fix: `map-minitest-spec-assertion-forms`
(RFC 0122) resolves a spec spelling to the builtin NAME and reuses `RAILS_MAP`,
rather than adding a second kind map that can drift.

## Acceptance criteria

- Extend the `CanonicalKind` vocabulary with the kinds needed to compare the
  Rails-helper-twin groups above — at minimum a query-count kind
  (`assert_queries_count` / `assert_no_queries` / `assert_queries_match`) and a
  difference/change kind (`assert_difference` / `assert_no_difference` /
  `assert_changes`). Each new kind is justified by a Rails helper that exists on
  BOTH sides; a kind that only ever appears on one side is not added.
- Map the Ruby helper and its trails mirror onto the same kind, each entry
  citing the Rails `file:line` that defines the helper.
- Leave the mock family, `toHaveProperty`, `expectTypeOf`, `toBeTypeOf` and
  `toMatchSnapshot` unmapped, and say so in a comment — they have no Rails
  counterpart, and the unmapped report is the honest answer for them.
- **Report the effect on all five gated packages' marks, before and after.**
  Newly-comparable assertions will surface real divergence, exactly as
  `must_be_like` did for arel: arel's `value` count went 17 → 79 the moment 326
  assertions became comparable. Expect counters to RISE.
- The mark file `scripts/test-compare/assertion-mismatch-mark.json` is
  only-shrink. If a counter rises, that is a burndown to file as follow-up
  stories, **not** a mark to raise — RFC 0122's one-time arel `value`
  correction was scoped to that RFC and is not precedent here. Split the work
  so each PR either lowers a counter or leaves it flat.
- `pnpm parity:test:assertions` is green.
- No test is renamed or reworded to make a number move.
