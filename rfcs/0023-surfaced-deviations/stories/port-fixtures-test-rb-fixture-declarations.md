---
title: "Port fixtures_test.rb's fixture declarations and drop fixtures.test.ts from the expected-fixtures ratchet"
status: draft
updated: 2026-07-27
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

Surfaced by PR #5403 (RFC 0064 bucket-D disposition). That PR moved
`test-helpers/define-fixtures.ts` to `packages/activerecord/src/fixtures.ts`,
its Rails home (`activerecord/lib/active_record/fixtures.rb:527`, with
`create_fixtures` at `:595`). Its test moved alongside as
`packages/activerecord/src/fixtures.test.ts`.

At the old invented name the `expected-fixtures` lint rule mapped the test file
to no Rails test at all, so it was silently unchecked. Under the Rails name it
now maps to `vendor/rails/activerecord/test/cases/fixtures_test.rb`, and the
rule immediately fired: our file declares none of the 34 fixture sets that
Rails' `FixturesTest` dereferences —

```text
accounts, admin/accounts, admin/randomlyNamedA9, admin/randomlyNamedB0,
admin/users, badPosts, books, bulbs, categories, computers, courses,
cpkAuthors, cpkBooks, cpkOrderAgreements, cpkOrders, cpkReviews, deadParrots,
developers, dogs, doubloons, funnyJokes, items, liveParrots, organizations,
otherBooks, otherComments, otherDogs, otherPosts, parrots, pirates,
randomlyNamedA9, ships, topics, trafficLights, treasures
```

PR #5403 added `packages/activerecord/src/fixtures.test.ts` to
`eslint/expected-fixtures-exclude.json` — the documented ratchet for exactly
this ("Files currently lacking it are tracked in
`eslint/expected-fixtures-exclude.json` and ratcheted down as porters
migrate", `eslint.config.mjs:400-403`). That was the right move to keep the
rename mechanical, but it parks a real porting gap rather than closing it.

The gap is genuine, not a mapping artefact: `fixtures_test.rb` IS the Rails
test for `fixtures.rb`, and our `fixtures.test.ts` is only a partial port of it.

## Acceptance criteria

- `packages/activerecord/src/fixtures.test.ts` declares its fixture sets via
  `fixtures({ ... })` (the endgame surface) so `expected-fixtures` passes on
  its own terms.
- `packages/activerecord/src/fixtures.test.ts` is REMOVED from
  `eslint/expected-fixtures-exclude.json` — the point of the story is to
  ratchet the list down by one, not to keep the entry.
- Test names match `fixtures_test.rb` verbatim; do not reword. Use
  `pnpm rails:find` to map each name to its `file:line` before porting.
- Any fixture set or model the canonical schema lacks is added to the canonical
  schema (`test-helpers/test-schema.ts` + `test-helpers/fixtures/`), mirroring
  `vendor/rails/activerecord/test/schema/schema.rb` and
  `test/fixtures/`. Do not invent tables.
- `pnpm parity:test` delta for `fixtures_test.rb` is non-negative;
  `pnpm parity:schema` / `pnpm parity:fixtures` output unchanged unless
  canonical additions are part of the work, in which case the delta is
  positive.

## Notes

Scope check before starting: `fixtures_test.rb` is large. If porting the whole
file exceeds the LOC ceiling, ship the portion that fits, keep the exclude
entry until the last case lands, and register the remainder as a follow-up
story rather than fanning out PRs.
