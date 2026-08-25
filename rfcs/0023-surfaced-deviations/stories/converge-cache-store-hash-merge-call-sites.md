---
title: "converge cache/store.ts's Hash#merge call sites so the call-set gate sees them"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Surfaced in PR #6445 (`port-cache-retrieve-pool-options`).

`retrieve_pool_options` merges the caller's hash over the defaults:

```ruby
# activesupport/lib/active_support/cache.rb:215
pool_options = DEFAULT_POOL_OPTIONS.merge(pool_options)
```

`packages/activesupport/src/cache/store.ts` spells this as an object spread, so
the call-set gate flags the missing `merge`; it ships with a
`@missingRailsCall merge` tag at the call site. The same file already carries a
baselined row for the same call — `merged_options` / `merge` in
`scripts/api-compare/call-mismatches-exclude/activesupport/cache.json` — which
is inherited RFC 0047 seed debt.

## Converged shape

Decide once how a Ruby `Hash#merge` is spelled in trails so the call-set gate
sees it, then apply it to both sites in `cache/store.ts`: `retrieve_pool_options`
(cache.rb:215) and `merged_options` (cache.rb:861-888). Converging both retires
the `@missingRailsCall` tag and deletes the baselined `merged_options` row by
hand (only-shrink; no reseed).

## Acceptance criteria

- [ ] Both `merge` sites in `cache/store.ts` call the same ported counterpart.
- [ ] The `@missingRailsCall merge` tag is removed.
- [ ] The `merged_options` / `merge` row is deleted from the exclude shard and
      `pnpm parity:api:calls` is green.
