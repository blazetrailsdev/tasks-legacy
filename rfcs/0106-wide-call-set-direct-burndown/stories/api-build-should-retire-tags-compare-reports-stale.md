---
title: "parity:api:build should retire a tag compare.ts reports stale"
status: done
updated: 2026-08-22
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: 6886
claim: "2026-08-22T21:34:59Z"
assignee: "api-build-should-retire-tags-compare-reports-stale"
blocked-by: null
closed-reason: null
---

## Context

PR #6881 closed the silent-loss half of
`parity-api-build-must-not-drop-harvested-tags`: `reconcileFileText`
(`scripts/api-compare/build.ts`) now defaults `expected` to the declaration's own
existing tagged calls when the artifact carries NO expectation for it, instead of
an empty set that read every tag as satisfied and deleted it.

That is deliberately conservative, and it leaves a gap in the other direction.
`build.ts` can no longer retire a tag on a declaration `compare.ts` does not
match to a Ruby method, even when the tag IS stale — the run prints
a `preserved` line naming the declaration and the call, and moves on. The
stale half is still detected, by `staleCallTags`
(`scripts/api-compare/compare.ts:1419`, surfaced through `parity:api:calls`'s
"STALE @missingRailsCall tag(s)" arm), but retiring one is a hand edit today.
PR #6881 hit exactly that: `benchmarkable.ts`'s `@missingRailsCall logger` had gone
stale and was deleted by hand as a drive-by.

The two facts live one module apart and nothing joins them: the gate KNOWS the
tag is stale, and the writer that could remove it does not ask.

## Converged shape

Teach `build.ts` to consume the artifact's `staleTags` (the same
`StaleCallTag[]` `compare.ts` already writes) as a THIRD input alongside
`mismatches` and `suppressed`. A preserved tag whose (file, decl, call) appears
in `staleTags` is a confirmed convergence: drop it and report it on the
`DROPPED …` line, which already states the call is no longer flagged. A
preserved tag NOT in `staleTags` stays preserved, exactly as now.

Note the artifact's stale rows are keyed by tag site (file + comment), not by
the `tsClass`/`tsName` expectation key `buildExpectations` builds — check
`staleCallTags`' row shape before keying off it.

## Acceptance criteria

- [ ] `parity:api:build` retires a tag `compare.ts` reports stale, even where the
      declaration carries no expectation, and says so on the DROPPED line.
- [ ] A preserved tag with no stale row is still preserved (the #6873 shape
      `build.test.ts` already covers must stay green).
- [ ] Regression test: a fixture with one stale-reported tag and one merely
      unmatched tag — the first is dropped, the second survives.
