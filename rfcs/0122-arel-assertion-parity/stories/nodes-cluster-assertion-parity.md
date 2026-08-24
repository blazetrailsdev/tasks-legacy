---
title: "Converge the nodes/* assertion parity tail"
status: ready
updated: 2026-08-24
rfc: "0122-arel-assertion-parity"
cluster: null
packages: ["arel"]
deps: ["map-minitest-spec-assertion-forms"]
deps-rfc: []
est-loc: 240
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

RFC 0122's per-file burndown. Rails file(s): `nodes/*_test.rb, attributes_test.rb, nodes_test.rb, collectors/*_test.rb`
(under `vendor/rails/activerecord/test/cases/arel/`). trails file(s): `packages/arel/src/nodes/*.test.ts, packages/arel/src/attributes.test.ts, packages/arel/src/nodes.test.ts, packages/arel/src/collectors/*.test.ts`.

Measured with `map-minitest-spec-assertion-forms` applied
(`pnpm parity:test -- --assertions --missing --package arel`): **44 assertion-kind
mismatches** remain in this cluster, plus their share of arel's 178
assertion-count and 79 assertion-value mismatches.

The long tail: no file above 5 mismatches (sql_literal 5, ascending 5, descending 5, case 5, unary_operation 4, window 3, as 3, …). Mostly the extra-`toBeInstanceOf` and extra-`toBe` classes rather than `must_be_like`. Split into a second PR if it exceeds the LOC ceiling — file the remainder as a new story, do not fan out.

The dominant class repo-wide is a Rails full-string `must_be_like` ported as a
substring `toContain`, often split into two — which inflates the count
dimension as well. Canonical example:
`vendor/rails/activerecord/test/cases/arel/select_manager_test.rb:15-23`
asserts `_(manager.to_sql).must_be_like %{ SELECT id FROM "users" }` where
`packages/arel/src/select-manager.test.ts:35` asserts
`expect(mgr.toSql()).toContain("SELECT id")`. `must_be_like` squeezes
whitespace (`helper.rb:10-13`), so the faithful port is a single `toEqual` on
the squeezed SQL string, not a substring check.

## Acceptance criteria

- Every mismatch in these files is triaged into exactly one bucket and handled:
  - **real divergence** — mirror the Ruby: a `toContain` standing in for a
    `must_be_like` becomes one `toEqual` on the whitespace-squeezed SQL from
    the Rails source; an assertion checking the wrong literal takes the Rails
    literal.
  - **legitimate trails-only extra** — no Rails counterpart. Move it to a
    `.trails.test.ts` sibling. Do not delete rigour, and do not leave it
    inflating a mirrored test's counts.
  - **further tooling gap** — fix it in
    `scripts/test-compare/assertion-kinds.ts` with a one-line justification
    citing both sides' semantics, and report the effect on all five packages'
    marks. Never a per-file workaround.
- **No test name is renamed or reworded.** Names are how `parity:test` matches.
- The touched tests pass under `pnpm vitest run`.
- `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry is tightened
  by the amount this story converged, on each of the three dimensions. The mark
  is only-shrink here — the one-time `value` correction is scoped to
  `map-minitest-spec-assertion-forms`.
- `pnpm parity:test:assertions` is green, and the other four packages' marks
  are unchanged.
