---
title: "Disambiguate the two exported association() functions"
status: draft
updated: 2026-08-02
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two exported functions named `association` return different objects, and
nothing in the name or signature says which you get:

- `association(record, assocName)` in
  `packages/activerecord/src/associations.ts:2040` returns the user-facing
  `CollectionProxy` (from `record._collectionProxies`).
- `association(this, name)` in
  `packages/activerecord/src/associations/instance-methods.ts:130` — the one
  bound as `Base#association`, mirroring Rails' `ActiveRecord::Associations#association`
  (`activerecord/lib/active_record/associations.rb`) — returns the OO
  `Association` instance from `record._associationInstances`.

Rails has exactly one `association(name)`, returning the `Association`. The
trails split is a real trap: `CollectionProxy` itself calls
`this._record.association(this._assocName)` at roughly a dozen sites
(`collection-proxy.ts` ~1089, 1122, 1147, 2109, 2281, 2413, 2614, 2727, 2807,
3336, 3458, 3572) and casts the result to an ad-hoc structural type each time.
Whether that resolves to the OO association or back to the proxy depends on
which module the caller imported from — and a call that resolved to the proxy
would infinitely recurse for a delegating method. This was the single riskiest
step while writing `CollectionProxy#find`'s delegation (PR #5910); it took
reading both function bodies to establish which one `this._record.association`
is.

## Acceptance criteria

- The two functions no longer share the bare name `association`. The
  Rails-named one (`Base#association`, returning the `Association`) keeps the
  name; the proxy-returning one in `associations.ts` is renamed to say what it
  returns (e.g. `collectionProxyFor`) and its call sites updated.
- `CollectionProxy`'s repeated
  `this._record.association(this._assocName) as unknown as { ... }` casts route
  through one private accessor returning the OO `CollectionAssociation`, so the
  structural cast is written once rather than a dozen times.
- No behaviour change: this is a naming/typing convergence, and the existing
  association suites stay green.
