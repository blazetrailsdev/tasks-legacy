---
title: "Converge table, factory_methods and attributes/math assertion parity"
status: done
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: ["arel"]
deps: ["map-minitest-spec-assertion-forms"]
deps-rfc: []
est-loc: 170
priority: null
pr: 7029
claim: "2026-08-25T12:46:55Z"
assignee: "converge-reverse-merge-bang-key-presence"
blocked-by: null
closed-reason: null
---

## Context

RFC 0122's per-file burndown. Rails file(s): `table_test.rb, factory_methods_test.rb, attributes/math_test.rb`
(under `vendor/rails/activerecord/test/cases/arel/`). trails file(s): `packages/arel/src/table.test.ts, packages/arel/src/factory-methods.test.ts, packages/arel/src/attributes/math.test.ts`.

Measured with `map-minitest-spec-assertion-forms` applied
(`pnpm parity:test -- --assertions --missing --package arel`): **32 assertion-kind
mismatches** remain in this cluster, plus their share of arel's 178
assertion-count and 79 assertion-value mismatches.

`table_test.rb` carries much of arel's `must_be_kind_of` usage, so its residue after the mapping story is the extra-`toBeInstanceOf` class (`instanceOf rails 0 vs trails 1`) rather than the `toContain` one — expect relocation to `.trails.test.ts` to do more of the work here than rewriting.

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
- Use the shared `mustBeLike` normalizer from
  `packages/arel/src/test-helpers/must-be-like.ts`, which mirrors Rails putting
  it in `vendor/rails/activerecord/test/cases/arel/helper.rb:10-13`. It is a
  string normalizer inside a native `expect(...).toBe(...)`, NOT an assertion
  wrapper — `expect(mustBeLike(visitor.compile(ast))).toBe(mustBeLike(\`…\`))`—
so the extractor reads the terminal`toBe`with no special-casing. If that
file does not exist yet, hoist it there as part of this story from its
current local definition at`packages/arel/src/select-manager.test.ts:15`,
  and leave no second copy behind. Do not redefine it per test file.
- **No test name is renamed or reworded.** Names are how `parity:test` matches.
- The touched tests pass under `pnpm vitest run`.
- `scripts/test-compare/assertion-mismatch-mark.json`'s arel entry is tightened
  by the amount this story converged, on each of the three dimensions. The mark
  is only-shrink here — the one-time `value` correction is scoped to
  `map-minitest-spec-assertion-forms`.
- `pnpm parity:test:assertions` is green, and the other four packages' marks
  are unchanged.
