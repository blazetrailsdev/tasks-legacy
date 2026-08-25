---
title: "ActiveSupport::JSON.decode: port parse_json_times date coercion and quirks-mode parity"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`ActiveSupport::JSON.decode` in trails
(`packages/activesupport/src/json.ts`) is `JSON.parse` plus a
strip-comments-and-retry fallback added by PR #5351.

Rails' version
(`vendor/rails/activesupport/lib/active_support/json/decoding.rb:22-30`) does
two things trails does not:

1. `::JSON.parse(json, quirks_mode: true)` — the comment tolerance PR #5351
   emulates is one consequence of quirks mode; other quirks-mode
   permissiveness (bare scalars, `NaN`/`Infinity`) is not covered by the
   comment-stripping retry.
2. `convert_dates_from(data)` when `ActiveSupport.parse_json_times` is set —
   trails has neither the `parse_json_times` config nor any date coercion on
   decode, so `DATE_REGEX` / `DATETIME_REGEX` strings come back as strings
   regardless of the setting.

## Acceptance criteria

- Add the `ActiveSupport.parse_json_times` config accessor and the
  `convert_dates_from` traversal, matching `decoding.rb:22-30` and its
  `DATE_REGEX` / `DATETIME_REGEX` constants.
- Decide explicitly what the trails analogue of `quirks_mode: true` is and
  either implement it or justify the comment-stripping retry as the converged
  form at the call site.
- Port the corresponding Rails cases from
  `vendor/rails/activesupport/test/json/decoding_test.rb` (test names verbatim).
