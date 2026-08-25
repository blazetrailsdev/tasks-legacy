---
title: "ShardSelector#call does not build an ActionDispatch::Request"
status: draft
updated: 2026-08-25
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Middleware::ShardSelector#call` wraps the rack env before the
resolver sees it:

```ruby
# activerecord/lib/active_record/middleware/shard_selector.rb:40-48
def call(env)
  request = ActionDispatch::Request.new(env)
  shard = selected_shard(request)
  set_shard(shard) { @app.call(env) }
end
```

trails' `call()` (`packages/activerecord/src/middleware/shard-selector.ts`) is
handed the request object itself and constructs nothing, so the resolver block
receives a different object than Rails' block does — a resolver written against
Rails' `request.params` / `request.headers` surface does not port across.

`ActionDispatch::Request` now exists in trails
(`packages/actionpack/src/action-dispatch/http/request.ts`), so the blocker is
no longer "unported". What blocks it is packaging: Ruby resolves the constant
when `call` runs, so `activerecord.gemspec` declares no actionpack dependency,
whereas an ESM `import` is eager and would make actionpack a hard dependency of
activerecord that Rails does not have. The settled trails answer for a
call-time constant is the zero-import slot module (CLAUDE.md, "Call-time
constant resolution"), and this is a cross-package instance of it.

PR #7009 retired the `call -> new` call-set exclude row in favour of a
CONVERGEABLE `@missingRailsCall` receipt at the call site; this story is what
retires the receipt.

## Acceptance criteria

- [ ] `call(env)` builds an `ActionDispatch::Request` from the env and hands
      that to `selectedShard`, as `shard_selector.rb:41` does.
- [ ] activerecord gains no eager `import` of `@blazetrails/actionpack` and no
      `dependencies` entry for it — the constant is reached at call time.
- [ ] The `@missingRailsCall new` receipt on `ShardSelector#call` is deleted.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
