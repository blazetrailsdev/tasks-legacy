---
title: "Fragment caching drops content.to_str and never forwards options to exist?/delete/delete_matched"
status: claimed
updated: 2026-08-24
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 120
pr: null
claim: "2026-08-24T09:21:48Z"
assignee: "association-cache-holds-only-association-instances"
blocked-by: null
closed-reason: null
---

## Context

Surfaced converging the `key` parameter reassignment in
`abstract-controller/caching/fragments.ts` (PR #6378). Two divergences in that
file were out of scope for a naming PR and are still standing, three of them as
`kind: "args"` rows in
`scripts/api-compare/call-mismatches-exclude/abstractcontroller/caching/fragments.json`.

**1. `write_fragment` drops `content.to_str`.**
Rails (`vendor/rails/actionpack/lib/abstract_controller/caching/fragments.rb:80-89`):

```ruby
instrument_fragment_cache :write_fragment, key do
  content = content.to_str
  cache_store.write(key, content, options)
end
```

`packages/actionpack/src/abstract-controller/caching/fragments.ts:102-114`
writes `content` straight through. `to_str` is Ruby's _implicit_ string
conversion — it raises `NoMethodError` for anything that is not string-like,
which is the guard Rails relies on to reject a non-string fragment body.

**2. `cache_store.exist?` / `delete` / `delete_matched` never receive `options`.**
Rails passes `options` to all three (`fragments.rb:110,137,139`). The port
drops the argument and marks the parameter `_options`, with an inline follow-up
note at `fragments.ts:124-128`: the trails `CacheStore` interface in
activesupport does not accept options on those methods. Rails' own
`ActiveSupport::Cache::Store` does
(`vendor/rails/activesupport/lib/active_support/cache.rb` — `exist?(name,
options = nil)`, `delete(name, options = nil)`, `delete_matched(matcher,
options = nil)`), so the interface is the thing that diverged, not the caller.

## Acceptance criteria

- `CacheStore.exist`, `CacheStore.delete` and `CacheStore.deleteMatched` in
  activesupport take the optional `options` argument Rails' `Cache::Store`
  takes, and every implementation in the repo accepts it.
- `fragmentExist` and `expireFragment` pass `options` through, and the `_`
  prefix plus the follow-up comment at `fragments.ts:124-128` are deleted.
- `writeFragment` performs Rails' `content.to_str` step inside the instrument
  block, raising for a non-string-like body rather than silently caching it.
- The three `kind: "args"` rows in
  `call-mismatches-exclude/abstractcontroller/caching/fragments.json` are
  deleted by hand (only-shrink, no reseed); `pnpm parity:api:calls:args` green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
