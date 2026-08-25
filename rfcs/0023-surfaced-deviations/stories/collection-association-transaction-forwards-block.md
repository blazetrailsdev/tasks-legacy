---
title: "CollectionAssociation#transaction forwards its block instead of re-wrapping it"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`CollectionAssociation#transaction`
(`packages/activerecord/src/associations/collection-association.ts:449-457`)
mirrors Rails' `reflection.klass.transaction(&block)`
(`vendor/rails/activerecord/lib/active_record/associations/collection_association.rb:499-501`),
but does not pass the block through: it re-wraps it,

```ts
return klass.transaction(() => Promise.resolve(block()));
...
return Promise.resolve(block());
```

PR #6442 introduced the wrapper when `concat` started handing `transaction` a
`Promise<R> | R` block (`concat_records` no longer always owes I/O). The wrapper
exists only because `Base.transaction`'s TS signature demands a
`() => Promise<R>`; at runtime it awaits whatever the block returns, so a
synchronous block already works.

The fallback arm (`Promise.resolve(block())` when the klass has no `transaction`)
is a second trails-only shape — Rails has no klass-less path here at all.

## Converged shape

Widen `Base.transaction`'s block parameter to `() => Promise<R> | R` (it already
handles both) and pass `block` straight through, so the ported body reads
`klass.transaction(block)` exactly as Rails does. Then re-check whether the
klass-less fallback is reachable at all; if it is not, drop it.

## Acceptance criteria

- [ ] `transaction` forwards `block` with no wrapping closure.
- [ ] `pnpm parity:api:calls:args` has no new row for this call site.
- [ ] `packages/activerecord/src/associations` green on all adapter lanes.
