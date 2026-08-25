---
title: "Duration#iso8601 inlines what Rails delegates to ISO8601Serializer"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails builds the ISO8601 form of a `Duration` through a separate serializer
object: `iso8601(precision: nil)` is
`ISO8601Serializer.new(self, precision: precision).serialize`
(`vendor/rails/activesupport/lib/active_support/duration.rb:473-475`), with the
class in `vendor/rails/activesupport/lib/active_support/duration/iso8601_serializer.rb`.

`packages/activesupport/src/duration.ts`'s `iso8601` inlines the whole string
build instead. The divergence was invisible until PR #5353 added
`Messages::Codec#serialize`, which put `serialize` into the api-compare
wide-call population; the `iso8601` -> `serialize` pair is now baselined in
`scripts/api-compare/call-mismatches-wide-exclude/activesupport/duration.json`
(the `iso8601` -> `new` half was already baselined before).

## Acceptance criteria

- Port `duration/iso8601_serializer.rb` to
  `packages/activesupport/src/duration/iso8601-serializer.ts`.
- Reduce `Duration#iso8601` to constructing it and calling `serialize`.
- Drop both `iso8601` entries from the duration wide-call exclude file.
- Existing duration tests keep passing; `parity:api --package activesupport`
  non-negative.

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
