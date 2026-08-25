---
title: "Sweep canonical model scopes and ported tests for string-vs-symbol order args"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while fixing `relation-order-string-arg-stays-bare` (PR #4952).

Now that a **string** order arg stays bare and only **Symbol**/**Hash** args
qualify to the table, the arg _type_ used in a canonical model scope or ported
test is semantically load-bearing — it decides whether the emitted SQL is
`ORDER BY id` or `ORDER BY "authors"."id"`. A bare column is ambiguous the
moment the relation joins.

PR #4952 fixed the instances that broke CI, and confirmed the models are a
genuine mix:

- `Author#goodRatings` / `#noJoinsGoodRatings` used `q.order("id")` where Rails
  uses `order(:id)` (author.rb:56-66) — had to become the qualified hash form,
  because the through-join made bare `id` ambiguous.
- `Author#favoriteAuthors` uses `q.order("name")` and Rails also uses
  `order("name")` (author.rb:156) — correctly left bare.

The remaining bare-string order scopes were not individually diffed against
Rails; they pass today only because their associations happen to be
single-table. Examples to check (non-exhaustive, from
`packages/activerecord/src/test-helpers/models/`):
`company.ts` (~15 `q.order("id")` scopes), `company-in-module.ts`,
`author.ts:230,532,659,661`, `post.ts:279,283,806`, `car.ts:87,95,101`,
`developer.ts:353,354`, `reply.ts:29`.

Same applies to ported tests that pass a bare string where the Rails test uses
a symbol; #4952 converted the ones that failed (`finder.test.ts`,
`relations.test.ts`, `calculations.test.ts`, `cascaded-eager-loading.test.ts`,
`reserved-word.test.ts`, `order.test.ts`) but did not sweep the rest.

## Acceptance criteria

- [ ] Each bare-string order arg in `test-helpers/models/**` is diffed against
      its Rails counterpart in `vendor/rails/activerecord/test/models/` and
      converted to the symbol/hash form wherever Rails uses a symbol; left as a
      string wherever Rails uses a string.
- [ ] Same sweep for ported test files: the order arg type matches the Rails
      test verbatim.
- [ ] No test name is changed (parity:test matching).
- [ ] Full AR suite green on sqlite + PG + MySQL/MariaDB.
