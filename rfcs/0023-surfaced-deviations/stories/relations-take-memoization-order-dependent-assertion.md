---
title: "De-flake RelationTest where with take memoization: unordered take() makes the id assertion PG-order-dependent"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`RelationTest > "where with take memoization"`
(`packages/activerecord/src/relations.test.ts`, the
`expect(firstPost!.id).not.toBe(thirdPost!.id)` assertion) reds the
`Active Record PostgreSQL Tests (2)` shard intermittently. Observed on
PR #6410's run 31604619856 as `expected 15 not to be 15`; the same commit
passes on the PG adapter when the file is run in isolation.

The test creates 5 posts titled "0".."4", then:

```ts
const firstPost = await postsRel.take(); // no ORDER BY
const thirdPost = await postsRel.where({ title: "3" }).take(); // no ORDER BY
```

Neither `take()` carries an ORDER BY, so `firstPost` is whatever row the PG
heap returns first. Inside a full `--shard=2/2` run — after earlier tests have
churned `posts` and freed heap pages — that can be the just-inserted "3" row,
making both ids equal and the assertion fail. SQLite and MySQL return a
stable-enough order for it to hold, so the flake is PG-only.

The assertion's real subject is Rails' relation memoization: `postsRel.take()`
must not memoize a result that `postsRel.where(...).take()` then reuses. Row
identity is incidental to that.

## Converged shape

Make the two `take()` calls deterministic so the memoization assertion is the
only thing under test — e.g. order the base relation before the first `take()`,
or assert on the memoized relation's identity/`title` rather than on the row id.
The test name is Rails-matched by `parity:test`, so only the body may change:
do not rename it. Confirm the reworked body still fails against a deliberately
re-memoizing `take()` before landing.

## Acceptance criteria

- [ ] The assertion no longer depends on PG's physical row order.
- [ ] Test name unchanged.
- [ ] The body still catches the regression it exists for (verify it fails when
      `take()` is made to reuse the memoized relation).
- [ ] Green on all three adapter lanes, including inside a full sharded PG run.
