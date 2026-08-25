---
title: "Reconcile lock_version like Rails and delete reflectLockVersionBump"
status: ready
updated: 2026-07-27
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

`reflectLockVersionBump` (`packages/activerecord/src/associations.ts`) is a
trails-only helper that writes the incremented `lock_version` back into the
in-memory record after a counter-cache UPDATE. It was tagged `@internal` in PR
5359 because the investigation confirmed no Rails method to converge onto, but
the underlying deviation stands and is worth removing.

Rails needs no such step. `Locking::Optimistic#update_counters`
(`vendor/rails/activerecord/lib/active_record/locking/optimistic.rb`) merges
the locking column bump into the same UPDATE statement, and the in-memory
record is reconciled through the `_update_record` lock-version write rather
than by a separate reflect call.

The declaration carries a long coupling note that must be read before touching
it: callers invoke it immediately after `parent.incrementBang(...)`, and the
wired instance `incrementBang` is `Persistence#incrementBang`, which persists
via `this.constructor.updateCounters`. The separate
`Locking::Optimistic#incrementBang` in `locking/optimistic.ts` uses
`updateColumn` and is deliberately not wired for instance dispatch; if that
ever changes, the DB bump disappears and this sync changes meaning.

Its one external caller is
`packages/activerecord/src/associations/belongs-to-association.ts`, plus one
internal call site in `associations.ts`.

## Acceptance criteria

- The in-memory `lock_version` is reconciled the way Rails does it, rather than
  by a bespoke post-`incrementBang` reflect step.
- `reflectLockVersionBump` is deleted from `associations.ts` and its two call
  sites are updated; `associations.ts` novel extra count drops by one.
- Optimistic-locking and counter-cache suites pass with no test renames, and
  the counter-cache-with-locking behavior the helper was protecting stays
  covered.
