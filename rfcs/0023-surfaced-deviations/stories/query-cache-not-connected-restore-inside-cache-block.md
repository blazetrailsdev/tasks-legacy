---
title: "query-cache not-connected test restores the pool outside the cache block, Rails restores inside"
status: draft
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`vendor/rails/activerecord/test/cases/query_cache_test.rb:559-573`
(`test_cache_is_available_when_using_a_not_connected_connection`) places its
`ensure` _inside_ the `Task.cache do ... end` block:

```ruby
Task.cache do
  assert_queries_count(1) { Task.find(1) }
  assert_no_queries { Task.find(1) }
ensure
  ActiveRecord::Base.establish_connection(original_connection)
end
```

So the original connection is re-established _before_ `cache`'s own teardown
runs, meaning the cache-disable/clear at block exit acts on the restored pool.

`packages/activerecord/src/query-cache.test.ts` restores in a `finally` that
wraps the whole `Task.cache(...)` call instead, so teardown runs against the
not-connected pool and the restore happens after. PR #5445 (which removed this
test's `Base.configurations` workaround) left the placement as-is — it predates
that PR and was out of its scope.

Determine whether the ordering is observable in trails (which pool `cache`'s
completion disables/clears), and if so move the restore inside the block to
match Rails.

## Acceptance criteria

- [ ] Restore ordering in "cache is available when using a not connected connection" matches Rails' inside-the-block `ensure`, or the deviation is justified at the call site with the reason the JS analogue cannot express it.
- [ ] Test still passes on sqlite/pg/mysql lanes.
