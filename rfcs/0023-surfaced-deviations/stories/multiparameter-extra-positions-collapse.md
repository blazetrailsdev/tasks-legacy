---
title: "AcceptsMultiparameterTime: extra positions must collapse positionally like values_hash.sort"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Rails' `AcceptsMultiparameterTime` assembles multiparameter values via
`values_hash.sort.map!(&:last)` then `::Time.public_send(default_timezone, *values)`
(`vendor/rails/activemodel/lib/active_model/type/helpers/accepts_multiparameter_time.rb:45-46`).
Sorting and splatting means positions COLLAPSE: `{1=>2004, 2=>1, 3=>1, 7=>99}`
becomes `Time.utc(2004, 1, 1, 99)` — key 7's value lands in the 4th positional
slot (hour) and raises ArgumentError. A dense hash with keys 1..7 passes slot 7
as Time's 7th arg (usec).

Trails' `AcceptsMultiparameterTime#castFromMultiparameter`
(`packages/activemodel/src/type/helpers/accepts-multiparameter-time.ts:146`,
post-#5146) reads slots "1".."6" by key and silently ignores extra positions,
so the sparse-hash example casts to a valid midnight instead of raising, and a
7th slot (usec) is dropped.

Surfaced by Copilot review on #5146; deferred there as an exotic input shape —
the AR extractor produces keys from real form params, where positions > 6 on a
date/time column are pathological.

## Acceptance criteria

- `castFromMultiparameter` builds the argument list Rails-style: sort keys
  numerically, map to values, splat positionally (respecting the per-type
  `defaults` fill first), so extra positions shift into later Time slots
  rather than being dropped.
- `{1: 2004, 2: 1, 3: 1, 7: 99}` on a datetime type raises ArgumentError
  (99 lands in the hour slot); a dense 1..7 hash treats slot 7 as usec.
- Existing multiparameter suites (activerecord multiparameter-attributes,
  activemodel date/date-time/time, accepts-multiparameter-time-defaults)
  still pass.
