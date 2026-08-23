---
title: "duration-extracts-iso8601-serializer-and-parser"
status: claimed
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T12:12:26Z"
assignee: "duration-extracts-iso8601-serializer-and-parser"
blocked-by: null
closed-reason: null
---

## Context

`duration.rb:474` `ActiveSupport::Duration#iso8601` is
`ISO8601Serializer.new(self, precision: precision).serialize`, and
`duration.rb:144-147` `Duration.parse` is
`parts = ISO8601Parser.new(iso8601duration).parse!`. Rails extracts both into
`active_support/duration/iso8601_serializer.rb` and
`active_support/duration/iso8601_parser.rb`.

trails inlines both bodies into `iso8601` and `parse` in
`packages/activesupport/src/duration.ts`, so `parse`'s invalid-input guards
`throw new Error(...)` rather than raising through `ISO8601Parser::ParsingError`.
Three rows remain baselined in
`scripts/api-compare/call-mismatches-exclude/activesupport/duration.json`
(`iso8601 | new`, `iso8601 | serialize`, `parse | order:calculateTotalSeconds,constructor`);
RFC 0106 wave 5g deliberately left them rather than mint `@missingRailsCall`
receipts, because a receipt would ratify an inlining Rails does not do.

## Acceptance criteria

- [ ] `ISO8601Serializer` is extracted to
      `packages/activesupport/src/duration/iso8601-serializer.ts` with Rails'
      `initialize(duration, precision: nil)` / `serialize` decomposition, and
      `iso8601` calls it (duration.rb:473-475).
- [ ] `ISO8601Parser` is extracted to
      `packages/activesupport/src/duration/iso8601-parser.ts` with Rails'
      `parse!` and its `ParsingError`, and `Duration.parse` raises through it
      (duration.rb:144-147).
- [ ] All three baseline rows deleted, not reworded;
      `pnpm parity:api:calls:tighten activesupport/duration.json` if stale.
- [ ] `pnpm parity:api:calls`, `:args` and `:extra` green.
