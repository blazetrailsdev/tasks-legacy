---
title: "Messages::Metadata models time as Temporal.Instant, stranding five Rails Time calls"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 220
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activesupport/src/messages/metadata.ts` models every time value as a
`Temporal.Instant` rather than through trails' `Time` / `DateTime` analogues.
Because an `Instant` is absolute by construction and exposes none of the Ruby
`Time` API, five separate Rails calls have no receiver to land on and each
carries a `@missingRailsCall ... PERMANENT` receipt at its site:

| Rails                               | `metadata.rb` | trails                                                        |
| ----------------------------------- | ------------- | ------------------------------------------------------------- |
| `Time.now.utc >= parse_expiry(...)` | `:81`         | `Temporal.Instant.compare(currentTimeInstant(), expiry) >= 0` |
| `expires_at.utc`                    | `:100-105`    | the `Instant` used as-is                                      |
| `Time.now.utc.advance(seconds:)`    | `:100-105`    | `currentTimeInstant().add({ milliseconds })`                  |
| `Time.iso8601(expires_at)`          | `:113-118`    | `Temporal.Instant.from(...)`                                  |
| `Time.parse(expires_at)`            | `:113-118`    | `new Date(...)` then `fromEpochMilliseconds`                  |

PR #6984 retired the two `utc` rows from
`scripts/api-compare/call-mismatches-exclude/activesupport/messages/metadata.json`
(taking activesupport to 0 `kind: "set"` rows for RFC 0106), but converged them
to receipts rather than to calls, because routing one call through the `Time`
analogue would replace the file's time type rather than converge a call. That
whole-file flip is what this story is for.

`DateTime#utc` is ported at
`packages/activesupport/src/core-ext/date-time/calculations.ts:340`
(`date_time/calculations.rb:184-191`), and `Time.utc` exists in
`@blazetrails/date`, so the receivers these calls need do exist.

Note the `pickExpiry` serialization branch: `expiry&.iso8601(3)`
(`metadata.rb:107`) is currently `toString({ smallestUnit: "millisecond" })`, so
the flip has to keep the wire format byte-identical — `Messages::Metadata` is a
serialization boundary and any drift breaks existing signed/encrypted messages.

## Acceptance criteria

- [ ] `metadata.ts` carries its time values as the trails `Time`/`DateTime`
      analogue, and the five calls above are real calls to the ported names.
- [ ] All five `@missingRailsCall ... PERMANENT` receipts are DELETED, not
      reworded — that is how this story is verified as done.
- [ ] The serialized `exp` value is unchanged: a message written before the
      change still verifies after it, on all three database lanes.
- [ ] `pnpm parity:api:calls` and `parity:api:calls:args` stay green;
      activesupport stays at 0 `kind: "set"` rows.
