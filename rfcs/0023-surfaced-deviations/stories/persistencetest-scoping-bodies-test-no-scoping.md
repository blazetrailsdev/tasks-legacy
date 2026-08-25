---
title: "Three PersistenceTest bodies carry Rails names but test no scoping and no assert_difference"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while shipping `fixture-teardown-has-no-delete-rails-deletes-at-next-load`
(PR #6273). Three `PersistenceTest` cases in
`packages/activerecord/src/persistence.test.ts` carry Rails test names but
bodies that test something else entirely:

- `destroy many` — Rails' `test_destroy_many`
  (`vendor/rails/activerecord/test/cases/persistence_test.rb:364-372`) finds
  `Client.find([2, 3])`, wraps the destroy in
  `assert_difference("Client.count", -2)`, asserts the returned records equal
  the found ones, and asserts they are all `frozen?`. Trails creates two
  `Topic`s and only checks a count.
- `class level destroy is affected by scoping` — Rails
  (`persistence_test.rb:1401-1411`) creates a `Reply`, pushes it onto
  `Topic.find(1).replies`, asserts `Topic.where("1=0").scoping { Topic.destroy(1) }`
  raises `RecordNotFound`, then asserts both records still exist. Trails
  destroys a topic it just created and checks a count — it exercises **no
  scoping at all**, so the name is a lie.
- `class level delete is affected by scoping` — same shape against
  `persistence_test.rb:1433-1440` (`Topic.where("1=0").scoping { Topic.delete(1) }`,
  then assert nothing raised for both records). Trails again exercises no
  scoping.

PR #6273 converged only the assertion _shape_ (absolute `count() === 0` → a
count delta, matching Rails' `assert_difference`) because the empty-table
assumption broke once fixture teardown stopped deleting. The bodies themselves
were left divergent as out of scope.

## Converged shape

Port all three bodies line for line from `persistence_test.rb`, including the
`scoping` blocks, the `Reply` creation and `replies <<` push, the
`assert_raise(RecordNotFound)` / `assert_nothing_raised` arms, and
`test_destroy_many`'s frozen-record and returned-records assertions. Declare the
fixture sets the Rails class declares (`Client`/`Topic`/`Reply` via the
`companies` and `topics` sets) rather than relying on records the test creates
itself — these tests reference fixture ids (`Client.find([2, 3])`,
`Topic.find(1)`) directly.

Test names stay exactly as they are.

## Acceptance criteria

- [ ] All three bodies match `persistence_test.rb:364-372`, `:1401-1411` and
      `:1433-1440`.
- [ ] The two scoping tests actually exercise `Model.where("1=0").scoping { ... }`.
- [ ] Fixture sets are declared rather than the tests creating their own
      subjects; no bare absolute-count assertion returns.
- [ ] `pnpm parity:test` delta non-negative (assertion counts/kinds improve).
- [ ] All AR lanes green.
