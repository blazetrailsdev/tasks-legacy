---
title: "Complete the Date._parse stand-in's zone table and grammar"
status: draft
updated: 2026-07-29
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

`packages/activemodel/src/type/date-time.ts` `parseTimeHash` is a hand-written
stand-in for Ruby's `Date._parse`, which `fallback_string_to_time` calls in
Rails (`vendor/rails/activemodel/lib/active_model/type/date_time.rb:67-76`).
It was verified differentially against real `Date._parse` over 27 strings in
PR #5567 (all agreeing), but that is a sample, not a proof.

Two known narrowings:

- `ZONE_OFFSETS` lists ~30 abbreviations. Ruby's table
  (`Date::Format::ZONES`, generated from `zonetab.list`) has hundreds.
  Unlisted abbreviations leave `offset` nil, which is `Date._parse`'s
  behavior for a genuinely unknown zone — so a missing entry is silently
  wrong rather than an error.
- `parseTimeHash` recognizes complete datetimes only. `Date._parse` also
  parses partial forms (time-only, `mday`-only, ordinal and week dates),
  which `new_time` then rejects on nil year — same end result today, but the
  equivalence is incidental, not designed.

## Acceptance criteria

- Zone abbreviation table generated from or checked against Ruby's
  `zonetab.list` rather than hand-listed, with ambiguous abbreviations
  resolved the way Ruby resolves them.
- A differential harness (Ruby `Date._parse` vs `parseTimeHash`) over a
  corpus large enough to cover the orderings and zone spellings, runnable on
  demand; document the residual gaps it reports.
- Any divergence it surfaces is either fixed or recorded with a Rails-source
  justification at the call site.
