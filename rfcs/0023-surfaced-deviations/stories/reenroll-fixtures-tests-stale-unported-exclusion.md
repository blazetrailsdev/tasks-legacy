---
title: "Re-enroll fixtures_test.rb/fixture_set/test_fixtures_test.rb: stale unported exclusion hides ~170 AR tests"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

PR #6466's exclusion audit found the largest single test-parity exclusion is
stale. `scripts/parity/unported-files/unscoped.ts:77-99` excludes:

- `fixtures.rb` / `testFile: "/fixtures_test.rb"` (153 Rails tests —
  `vendor/rails/activerecord/test/cases/fixtures_test.rb`)
- `fixture_set` / `testFile: "fixture_set/"` (14 tests,
  `vendor/rails/activerecord/test/cases/fixture_set/file_test.rb`)
- `test_fixtures.rb` / `testFile: "test_fixtures_test.rb"`

with the reason "The JS/TS ecosystem uses factories or ad-hoc Model.create
instead; Trails users won't ship YAML fixtures." That reason predates the
shipped port: `packages/activerecord/src/fixtures.ts`, `test-fixtures.ts`, and
the canonical fixture corpus (`test-helpers/fixtures/`) all exist, CLAUDE.md
names `fixtures({ ... })` as the canonical test surface, and `parity:fixtures`
gates that corpus. ≈170 real Rails tests are hidden from AR's test-parity
denominator (97.9% → ≈95.9% if reinstated).

Second bug in the same rows: the source pattern `"fixtures.rb"` is unanchored,
so it substring-matches `test_fixtures.rb` and
`encryption/encrypted_fixtures.rb` too (the known unported substring-shadow
trap). The anchored `/fixtures.rb` grammar exists
(`scripts/parity/unported-files/index.ts:38-43`) and is not used here.

Rails source: activerecord/test/cases/fixtures_test.rb,
activerecord/lib/active_record/fixtures.rb,
activerecord/lib/active_record/test_fixtures.rb. Converged shape: these rows
leave the unported registry; the test files enroll through the normal
test-compare flow (permanent-skip stubs already hold Rails test names
verbatim for unported files); remaining genuinely-unportable cases (ERB
preprocessing arms, Marshal) get case-level `tests:` exclusions with accurate
reasons instead of whole-file rows.

Related: draft story `port-fixtures-test-rb-fixture-declarations` (0023)
covers porting declarations; this story is the registry re-enrollment and
pattern anchoring.

## Acceptance criteria

- The three registry rows above are removed (or narrowed to case-level
  exclusions with reasons that are true today).
- Any remaining `fixtures`-family pattern is anchored so it cannot shadow
  `test_fixtures.rb` / `encrypted_fixtures.rb`.
- `pnpm parity:test` counts `fixtures_test.rb` / `fixture_set/` /
  `test_fixtures_test.rb` in AR's denominator; deltas non-negative.
