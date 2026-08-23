---
title: "No CI job runs packages/date: 362 date-gem tests are green-by-absence"
status: claimed
updated: 2026-08-23
rfc: "0061-ci-failures"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: "2026-08-23T22:12:31Z"
assignee: "date-suite-is-not-run-by-any-ci-job"
blocked-by: null
closed-reason: null
---

## Context

No CI job runs `packages/date`. `grep -n "packages/date" .github/workflows/ci.yml`
finds it only inside the change-filter regexes (`AR_PKGS_RE`, `UNIT_TESTS_PKGS_RE`,
… at ci.yml:109-118) — never in a `pnpm vitest run` argument list. The Unit
Tests job (ci.yml:598-624) runs `packages/arel packages/activemodel
packages/activesupport packages/i18n` plus the enumerated `scripts/` files, and
the coverage job (ci.yml:731-746) enumerates a different set; `packages/date` is
in neither, nor in any leaf job.

So the whole date-gem suite — 13 files, 362 tests, including every
`test-date-*.test.ts` ported under this RFC — is green-by-absence on CI. A
change to `packages/date` triggers the Unit Tests job (its regex names the
package) but the job then runs no date test.

`scripts/ci-suite-coverage.test.ts` did not catch this: it only asserts a
ci.yml vitest filter exists for each `scripts/`, `eslint/` and `vendor/` test
file, not for package suites. PR #6949 hit the `scripts/` half of that guard
and registered its new script test; the package half has no guard at all.

## Converged shape

Add `packages/date` to the Unit Tests `pnpm vitest run` list (ci.yml:598-614) —
its regex already routes date changes to that job — and to the coverage job's
list if the suite belongs in the coverage number. Then widen
`scripts/ci-suite-coverage.test.ts` so a package directory containing
`*.test.ts` with no ci.yml vitest filter is a failure, the same way a
`scripts/` test file already is, so the next unrouted package suite reds
immediately instead of sitting silent.

## Acceptance criteria

- [ ] `packages/date` appears in a ci.yml `pnpm vitest run` argument list.
- [ ] `scripts/ci-suite-coverage.test.ts` fails when a package with tests has
      no ci.yml vitest filter, and passes on the repo as it then stands.
- [ ] The date suite's 362 tests are visibly run in a CI job's output.
