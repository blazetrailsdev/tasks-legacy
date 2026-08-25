---
title: "whole-number Float loses its decimal point in SQL literals (Ruby Float#to_s)"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced while fixing the `Query Parity (diff)` red in PR #6973 (story `red-f8be16b9`).

Five rows in `scripts/parity/pipeline/canonical/query-known-gaps.json` —
`ar-170`, `ar-173`, `ar-174`, `ar-176`, `ar-179` — are one deviation: a
whole-number Float loses its decimal point when serialized into SQL.

Rails, `activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:82`
(`quote`), has `when Numeric then value.to_s`, and Ruby's `Float#to_s` renders
`4.0` as `"4.0"`, `5.0` as `"5.0"`, `0.0` as `"0.0"`. JavaScript has one numeric
type, so `String(4.0)` is `"4"` and the decimal is dropped. The gap shows up
across every predicate shape that inlines a float literal:

- `where({ rating: 4.0 })` → `= 4` (Rails `= 4.0`) — ar-176
- `where({ rating: 0.0 })` → `= 0` (Rails `= 0.0`) — ar-179
- `where({ rating: [3.5, 4.0, 4.5] })` → `IN (3.5, 4, 4.5)` — ar-173
- `where({ rating: new Range(3.5, 5.0) })` → `BETWEEN 3.5 AND 5` — ar-170
- `where({ rating: new Range(3.0, 5.0, true) })` → `>= 3 AND < 5` — ar-174

## Converged shape

The decimal has to come from the **type**, not from the JS value: the column is
a float/decimal column, so the serializer for that type is what knows to render
`Float#to_s` semantics — at least one fractional digit — rather than
`String(n)`. `Number.isInteger(n) ? n.toFixed(1) : String(n)` is the whole of
Ruby's `Float#to_s` for the values these fixtures cover; route it through the
float type's SQL serialization so every predicate shape above inherits it
rather than patching each `where` arm.

## Acceptance criteria

- A whole-number Float serializes with its decimal point in equality, `IN`,
  inclusive-`BETWEEN` and exclusive-Range SQL.
- All five rows are deleted from
  `scripts/parity/pipeline/canonical/query-known-gaps.json` and ar-170/173/174/176/179
  pass.
- Verified with both sides of the query pipeline plus
  `pnpm tsx scripts/parity/pipeline/query/diff.ts --rails-dir ... --trails-dir ...`.
