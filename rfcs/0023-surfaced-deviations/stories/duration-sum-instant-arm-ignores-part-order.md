---
title: "duration-sum-instant-arm-ignores-part-order"
status: draft
updated: 2026-08-09
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

Surfaced during review of PR #6286 (RFC 0088 story
`activesupport-core-ext-date-calculations-on-date-class`), which converged the
`time.acts_like?(:date)` arm of `Duration#sum` onto Rails' insertion-ordered
`@parts`.

`ActiveSupport::Duration#sum`
(`vendor/rails/activesupport/lib/active_support/duration.rb:397-419`) applies
`@parts.each` in Ruby Hash insertion order — the order the parts were merged in
(`:270-272`) — so `1.second + 1.day` applies seconds first and `1.day +
1.second` applies days first. Around DST that changes the resulting instant.

PR #6286 made `Duration`'s part _key set_ carry that order
(`packages/activesupport/src/duration.ts`, `_partKeys`), and the Date receiver
(`applyDurationToDate`) now walks it. **The instant receiver does not.**
`applyDuration` (`packages/activesupport/src/duration.ts`, reached from
`Duration#since`/`#ago` via `applyDurationPreservingNs`) ignores the key set
entirely and applies a hardcoded largest-to-smallest sequence — years via
`setFullYear`, months via `setMonth`, integer weeks/days via `setDate`, then a
single millisecond sum for fractional weeks/days plus hours/minutes/seconds.

That is a different shape from `sum` in three ways: the order is fixed rather
than the merge order, the parts are applied in one lump rather than a fold, and
the sub-day parts go through `t.since(sign * number)` in Rails.

Out of scope for #6286, which was scoped to `core_ext/date/calculations.rb` and
kept `applyDuration` untouched.

## Acceptance criteria

- `Duration#since`/`#ago` over an instant receiver fold the parts in
  `_partKeys` order, matching `sum`'s `@parts.each`.
- The `:seconds` / `:minutes` / `:hours` arms route through the receiver's
  `since` and the rest through `advance`, as `duration.rb:404-414` does.
- A cover showing `1.second + 1.day` and `1.day + 1.second` differ over a DST
  boundary, and that both match Rails.
- `pnpm parity:api:calls` non-negative; the `duration.ts` call-mismatch rows for
  `sum` converge or stay put, never grow.
