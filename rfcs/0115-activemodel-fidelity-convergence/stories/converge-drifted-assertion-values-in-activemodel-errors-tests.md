---
title: "Converge the drifted assertion values in activemodel's mirrored errors tests"
status: done
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 7006
claim: "2026-08-24T20:39:29Z"
assignee: "converge-attribute-method-predicate-to-rails-body"
blocked-by: null
closed-reason: null
---

## Context

Splitting the TS-only extras out of `errors.test.ts` (PR #6999, story
`move-ts-only-extras-out-of-mirrored-activemodel-errors-test-file`) left the
file holding exactly Rails' 86 test names. With the interleaving gone,
`pnpm parity:test --package activemodel --assertions` shows three mirrored tests
in `errors_test.rb` whose bodies assert different literal values than the Rails
test of the same name. These are ports that drifted, not TS-only extras.

Read from `scripts/test-compare/output/convention-comparison.json`
(`valueMismatches` for `errors_test.rb`), rails vs trails:

1. `count calculates the number of error messages` — rails `n:1`, trails `n:2`.
   Rails (`vendor/rails/activemodel/test/cases/errors_test.rb:429-433`) adds ONE
   error to a `Person` and asserts `assert_equal 1, person.errors.count`; the
   port adds two errors to a bare `Errors` and asserts `2`.

2. `full_message returns the given message when attribute is :base` — rails
   `s:press the button`, trails `s:Something went wrong`. Rails
   (`errors_test.rb:515-518`) is
   `assert_equal "press the button", person.errors.full_message(:base, "press the button")`;
   the port substitutes its own message string.

3. `clear removes details` — rails `n:1`, trails `n:0`. Rails
   (`errors_test.rb:619-626`) adds `:invalid`, asserts
   `assert_equal 1, person.errors.details.count`, THEN clears and asserts
   `assert_empty person.errors.details`; the port drops the pre-clear assertion
   and only checks the post-clear count.

Two further tests in the same file — `clear errors` and
`size calculates the number of error messages` — were held at their pre-split
assertion sets in PR #6999 so the ratchet would not regress; they are the same
class of drift (bare `new Errors(null)` where Rails uses a `Person`) and should
be converged in the same pass.

## Acceptance criteria

- Each of the five named tests mirrors its Rails body: same subject (`Person`,
  not a bare `Errors` where Rails uses a model), same assertions in the same
  order, same literal expected values.
- Test names are NOT changed — only bodies.
- `activemodel`'s `assertion-value-mismatch` counter drops, and
  `npx tsx scripts/test-compare/lint-assertion-mismatches.ts` reports OK.
  Tighten `scripts/test-compare/assertion-mismatch-mark.json` DOWN for the
  converged amount; never raise it.
- `errors_test.rb` still reports OK=86, Extra=0.

## Notes

Rails' `Person` model is `vendor/rails/activemodel/test/models/person.rb`;
trails' errors tests currently define a local `Person extends Model` inline
(see `it("adding errors using conditionals with Person#validate!")`). Check
whether a shared test model already exists before adding another inline one.
