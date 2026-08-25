---
title: "compute_cache_version's loaded-branch max swallows nils where Ruby's Array#max raises"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
packages: []
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

# `compute_cache_version`'s loaded branch swallows nils where Ruby's `Array#max` raises

## Context

Rails, `activerecord/lib/active_record/relation.rb:476-479`:

```ruby
if loaded?
  size = records.size
  if size > 0
    timestamp = records.map { |record| record.read_attribute(timestamp_column) }.max
  end
```

`Array#max` orders through `<=>`. If any record's timestamp attribute is `nil`,
`nil <=> Time` is `nil` and Ruby raises
`ArgumentError: comparison of NilClass with Time failed`. A collection with a
NULL `updated_at` therefore blows up rather than producing a cache version.

trails' `computeCacheVersion` (`packages/activerecord/src/relation.ts`) spells
the `max` as a `reduce` — necessary, because a cast datetime attribute sits on
a `Temporal.Instant`, which has no relational operators, so `<=>` has to be
spelled as `Temporal.Instant.compare`. But the reduce also carries two arms
Ruby does not have:

```ts
if (max == null) return value;
if (value == null) return max;
```

so a nil timestamp is skipped and the max of the remaining values is returned.
Rails raises; trails silently answers a cache version computed from a subset.

This predates PR #6663 — that PR converged the four call rows on this method
and kept the existing nil handling rather than widening its scope, and the
review pass explicitly flagged it as pre-existing and out of that PR's scope.
Filing so it is not lost.

## Converged shape

Drop the two nil arms. Keep the `Temporal.Instant.compare` dispatch, which is
the genuine language accommodation (Ruby gets `<=>` from the type; TS has no
operator overloading), and let a `null` reach the comparison so it fails the
way `Array#max` does. Match the error: Ruby raises `ArgumentError` with
`comparison of NilClass with Time failed`, so a bare TypeError is not the
converged answer — CLAUDE.md's "same error class, same message string, same
raise site" applies.

Check the unloaded branch too while in here: it takes `MAX(col)` from SQL,
where NULLs are skipped by the aggregate itself, so the two branches genuinely
differ in Rails as well. Do not "fix" that difference — it is Rails' own.

## Acceptance criteria

- [ ] The loaded branch raises `ArgumentError` with Rails' message when a
      record's timestamp attribute is nil, verified against MRI (`ruby -e`).
- [ ] The unloaded branch is unchanged — SQL `MAX` still skips NULLs.
- [ ] A test covers the raise; it fails on the current baseline.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
