---
title: "Preloader::Association#initialize drops the preload_scope.empty_scope? arm of @associate"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: guard-parity
packages: []
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the preloader reflection-scope memo (PR #6399).

`Preloader::Association#initialize`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/association.rb:110`):

```ruby
@associate = associate_by_default || !preload_scope || preload_scope.empty_scope?
```

trails' `packages/activerecord/src/associations/preloader/association.ts:50`
drops the third arm:

```ts
this._associate = associateByDefault || preloadScope == null;
```

So a loader constructed with `associateByDefault: false` and a preload scope
that is present but EMPTY does not associate its records to owners, where Rails
does. `Preloader::ThroughAssociation` builds exactly such loaders
(`through_association.rb:69,78`: `associate_by_default: false` with a `scope:`),
so an empty preload scope reaching a through loader silently skips the
associate step.

Rails' `empty_scope?` is `relation.rb:1299-1301`; the trails analogue is
`Relation#isEmptyScope` (`relation.ts:6274`), already used by the sibling
guards in `buildScope` and `associateRecordsFromUnscoped` after #6399.

## Converged shape

```ts
this._associate = associateByDefault || preloadScope == null || preloadScope.isEmptyScope;
```

## Acceptance criteria

- [ ] `Preloader::Association`'s `_associate` carries all three Rails arms
      (`association.rb:110`).
- [ ] A regression test covers a loader built with `associateByDefault: false`
      and an empty preload scope, and fails on the pre-fix baseline.
- [ ] AR preloader / through / eager suites pass on all three adapter lanes.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
