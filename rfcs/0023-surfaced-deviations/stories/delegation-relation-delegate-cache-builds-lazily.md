---
title: "delegation-relation-delegate-cache-builds-lazily"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails builds every model's relation delegate class EAGERLY, from
`ActiveRecord::Delegation::DelegateCache#initialize_relation_delegate_cache`
(`activerecord/lib/active_record/relation/delegation.rb:32-41`), which runs at
`inherited` time and, per delegate kind, does:

```ruby
delegate = Class.new(klass) {
  include ClassSpecificRelation
}
include_relation_methods(delegate)
```

trails' `initializeRelationDelegateCache`
(`packages/activerecord/src/relation/delegation.ts`) only seeds the cache; the
per-model delegate subclass is constructed lazily on first use by
`buildDelegateClass` (delegation.ts:416), which is where
`includeRelationMethods` runs. So neither `include` nor
`include_relation_methods` is called from the body that mirrors
`initialize_relation_delegate_cache`, and both are carried there as
`@missingRailsCall` receipts (added by the RFC 0106 wave-5d head sweep, PR for
story `wave-5d-head-sweep`).

## Acceptance criteria

- [ ] `initializeRelationDelegateCache` builds each delegate class eagerly, as
      `initialize_relation_delegate_cache` does, including the
      `includeRelationMethods` call — or the lazy build is shown to be forced by
      a TypeScript-level fact and that fact is stated at the call site.
- [ ] The two `@missingRailsCall` receipts on `initializeRelationDelegateCache`
      in `packages/activerecord/src/relation/delegation.ts` are deleted.
- [ ] `pnpm parity:api:calls` green; SQLite/PG/MySQL lanes green.
