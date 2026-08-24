---
title: "read_fragment drops Rails' html_safe step on the cached result"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced landing #6975. `read_fragment`
(`vendor/rails/actionpack/lib/abstract_controller/caching/fragments.rb:94-101`)
ends its instrument block with

```ruby
result = cache_store.read(key, options)
result.respond_to?(:html_safe) ? result.html_safe : result
```

`packages/actionpack/src/abstract-controller/caching/fragments.ts`
(`readFragment`) returns the store result unchanged. The file header claims
"Rails' `.html_safe` step on `readFragment` results is a no-op here — trails has
no html-safe marker", but that is no longer true: `htmlSafe` and `SafeBuffer`
ship from `@blazetrails/activesupport` and `packages/actionview/src/flows.ts`
already uses them. So a cached fragment read back is not marked html-safe and
will be escaped by a caller that trusts Rails' contract.

PR #6975 converged the other two divergences in this method group
(`content.to_str` on `write_fragment`, and `options` forwarding to
`exist?`/`delete`/`delete_matched`); this one was outside its acceptance
criteria.

## Converged shape

`readFragment` applies the `html_safe` arm — the trails spelling being
`htmlSafe(...)` from activesupport — for a `to_s`-able result, and the stale
"no html-safe marker" sentence leaves the file header.

## Acceptance criteria

- [ ] `readFragment` returns an html-safe result where Rails does, and the raw
      value where Rails does.
- [ ] The file header no longer claims trails has no html-safe marker.
