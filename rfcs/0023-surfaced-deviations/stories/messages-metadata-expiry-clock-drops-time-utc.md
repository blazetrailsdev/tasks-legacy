---
title: "Messages::Metadata expiry clock reads a JS Date, dropping Rails' Time.now.utc calls"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

Surfaced by #6891, which ported `DateTime#utc`
(`vendor/rails/activesupport/lib/active_support/core_ext/date_time/calculations.rb:184-191`)
and so put `utc` into the call-set population for the first time. Two
pre-existing omissions in
`packages/activesupport/src/messages/metadata.ts` became visible and had to be
baselined
(`scripts/api-compare/call-mismatches-exclude/activesupport/messages/metadata.json`,
rows `extract_from_metadata_envelope` and `pick_expiry`, each `call: "utc"`).

Rails, `vendor/rails/activesupport/lib/active_support/messages/metadata.rb`:

- `:81` — `if hash["exp"] && Time.now.utc >= parse_expiry(hash["exp"])`.
- `:100-105` `def pick_expiry(expires_at, expires_in)` — `expires_at.utc`, and
  `Time.now.utc.advance(seconds: expires_in)` on the other arm.

trails reads the clock as a JS `Date` on this seam, which already names an
absolute instant, so `.utc` is a no-op with no receiver and the call is
dropped.

## Converged shape

Read the clock through `@blazetrails/date`'s `Time` (`packages/date/src/time.ts`),
whose `getutc` is ruby/time's `utc`/`getutc` pair on an immutable receiver, so
both bodies make the call Rails makes. Note the expiry comparison and
`advance(seconds:)` have to keep working on whatever seat the serializers
round-trip — check `Metadata::TIMESTAMP_SERIALIZERS` (`:107-109`) and the
`iso8601(3)` tail before changing the value type.

## Acceptance criteria

- Both bodies call `Time#utc` where Rails does.
- Both rows are DELETED from
  `scripts/api-compare/call-mismatches-exclude/activesupport/messages/metadata.json`
  by hand (only-shrink — never `--write`/reseed); the file becomes empty and is
  removed. Narrow any stale high-water mark with `pnpm parity:api:calls:tighten`.
- `pnpm parity:api:calls` green.
- `packages/activesupport/src/messages/metadata*.test.ts` green; Rails test
  names unchanged.
