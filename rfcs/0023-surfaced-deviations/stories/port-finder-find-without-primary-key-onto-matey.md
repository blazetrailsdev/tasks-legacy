---
title: "Port 'find without primary key' onto the existing Matey model"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/finder.test.ts` "find without primary key" (in the
converged FinderTest2 block, merged in #5274) still asserts only that
`Post.all().toSql()` contains `SELECT`. Rails'
`test_find_without_primary_key` (vendor/rails/activerecord/test/cases/finder_test.rb:1748)
is:

```ruby
assert_raises(ActiveRecord::UnknownPrimaryKey) { Matey.find(1) }
```

During #5274 this was left alone on the belief that trails had no `Matey`
model. It does: `packages/activerecord/src/test-helpers/models/matey.ts`,
already used by `signed-id.test.ts:15` and `token-for.test.ts:8`. So the port
is unblocked.

## Acceptance criteria

- "find without primary key" asserts `Matey.find(1)` rejects with
  `UnknownPrimaryKey` (verify trails raises that error class; if it raises
  something else, that mismatch is the finding and should be fixed or filed).
- Test name unchanged. `mateys` fixtures/canonical schema only.
- `finder.test.ts` stays green.
