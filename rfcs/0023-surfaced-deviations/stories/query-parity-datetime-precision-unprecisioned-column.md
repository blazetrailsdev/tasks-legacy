---
title: "quoted_date keeps sub-second precision for an unprecisioned DATETIME column"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while fixing the `Query Parity (diff)` red in PR #6973 (story `red-f8be16b9`).

`scripts/parity/pipeline/canonical/query-known-gaps.json` carries three rows —
`ar-01`, `ar-52`, `ar-65` — for one deviation: trails keeps the sub-second
component when serializing a Time/Date bind against a DATETIME column that
declares **no** precision, where Rails truncates to whole seconds.

Rails, `activerecord/lib/active_record/connection_adapters/abstract/quoting.rb:184-197`
(`quoted_date`), appends `"." + sprintf("%06d", value.usec)` only when
`value.usec > 0` — and the value reaching it has already been rounded by
`ActiveModel::Type::Helpers::TimeValue#apply_seconds_precision`
(`activemodel/lib/active_model/type/helpers/time_value.rb:24-35`), which is a
no-op `return value unless precision && value.respond_to?(:nsec)`. For a bare
`created_at DATETIME` the column carries no precision, so the cast value never
gains a sub-second part and `quoted_date` emits
`'2026-04-19 00:46:48'`. trails emits `'2026-04-19 00:46:48.677000'`.

This is **not** a dormant gap: the diff harness fails on `UNEXPECTED-PASS` as
well as on failure (`scripts/parity/pipeline/query/diff.ts:301,312`), so whenever
the workflow's `PARITY_FROZEN_AT` happens to land on a whole second all three
rows report `UNEXPECTED-PASS` and red the job. PR #854 previously "closed" one
of these rows for exactly that reason. It is a live intermittent CI red.

## Acceptance criteria

- The datetime serializer applies Rails' precision semantics: with no column
  precision, the serialized bind and the inlined SQL literal both truncate to
  whole seconds, mirroring `apply_seconds_precision` + `quoted_date`.
- `ar-01`, `ar-52` and `ar-65` pass on every `PARITY_FROZEN_AT`, whole-second
  or not, and their three rows are deleted from
  `scripts/parity/pipeline/canonical/query-known-gaps.json`.
- Verified by running both sides of the query pipeline with a sub-second AND a
  whole-second frozen-at:
  `PARITY_FROZEN_AT=... pnpm tsx scripts/parity/pipeline/run.ts --type=query --side={rails,trails}`
  then `pnpm tsx scripts/parity/pipeline/query/diff.ts --rails-dir ... --trails-dir ...`.
