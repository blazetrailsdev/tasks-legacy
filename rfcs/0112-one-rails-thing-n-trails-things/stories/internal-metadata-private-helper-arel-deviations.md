---
title: "InternalMetadata private helpers: bare .eq vs BindParam, memoized currentTime, dead _q"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: duplicate-bodies
deps: []
deps-rfc: []
est-loc: 50
pr: 6811
claim: "2026-08-21T11:40:36Z"
assignee: "hash-config-primary-resolves-via-global-configurations"
blocked-by: null
closed-reason: null
---

## Context

Three small deviations in `packages/activerecord/src/internal-metadata.ts`,
confirmed as pre-existing and out of scope during the #5329 review (flagged in
all three review passes as untouched-but-real). Bundled because they sit in the
same private section of one file and are individually trivial.

1. **`selectEntry` drops the BindParam wrapper.** Rails
   (`internal_metadata.rb:154-162`) builds the predicate as
   `sm.where(arel_table[primary_key].eq(Arel::Nodes::BindParam.new(key)))`, so
   the key travels as a bind. trails (`internal-metadata.ts:~223`) uses a bare
   `.eq(key)`, which inlines the value at compile time. Same rows today, but the
   bind path is never exercised — and a create+find roundtrip does not catch
   this, so any regression test needs to assert on the emitted SQL/binds.

2. **`createEntry` memoizes `current_time` where Rails calls it twice.** Rails
   (`internal_metadata.rb:130-140`) calls `current_time(connection)`
   independently for `created_at` and `updated_at`; trails computes `const now`
   once and reuses it for both columns. Rails' two calls can differ by
   sub-millisecond; the memoized version guarantees they are identical.
   Converging means deciding whether the difference is worth reproducing or
   whether the memoization should be justified at the call site instead.

3. **Dead `_q` helper.** `internal-metadata.ts:51-53` defines
   `private _q(name: string)` delegating to `quoteIdentifier`. It has no callers
   — left behind when the hand-built DDL was replaced by adapter-driven
   `createTable`. No Rails counterpart. Delete it.

## Acceptance criteria

- `selectEntry` wraps the key in a BindParam node as Rails does, with a test
  asserting the emitted SQL carries a placeholder rather than an inlined value
  (a create+find roundtrip is not sufficient).
- `createEntry` either calls `currentTime(connection)` once per timestamp
  column like Rails, or keeps the memoization with a justification at the call
  site.
- `_q` is deleted.
- Existing internal-metadata tests stay green; no test renames.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
