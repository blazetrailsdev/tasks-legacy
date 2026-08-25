---
title: "Duration.parse inlines what Rails delegates to ISO8601Parser"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails parses an ISO 8601 duration through a separate parser object:
`Duration.parse(iso8601duration)` is
`parts = ISO8601Parser.new(iso8601duration).parse!` then
`new(calculate_total_seconds(parts), parts)`
(`vendor/rails/activesupport/lib/active_support/duration.rb:144-147`), with the
class in
`vendor/rails/activesupport/lib/active_support/duration/iso8601_parser.rb` —
a `StringScanner`-driven parser that raises
`ActiveSupport::Duration::ISO8601Parser::ParsingError`.

`packages/activesupport/src/duration.ts`'s `parse` inlines the whole thing: a
chain of literal-string guards, three "more invalid patterns" regexes, and one
big component regex, each raising a plain `Error("Invalid ISO 8601 duration:
...")` rather than `ParsingError`.

Surfaced by #6693 (converge-duration-constructor-value-seat): with `parse` now
filling the value seat in Rails' order — `calculateTotalSeconds(parts)`, then
`new Duration(value, parts)` — the call gate still reports a mismatch, because
the inlined guards' `throw new Error(...)` read to the extractor as
`constructor` calls ahead of `calculateTotalSeconds`. That is the row
`parse -> order:calculateTotalSeconds,constructor` in
`scripts/api-compare/call-mismatches-exclude/activesupport/duration.json`; it
converges by moving the parsing (and its throws) out of `parse`.

Sibling of `duration-iso8601-serializer-extraction`, which does the same for
the output half.

## Acceptance criteria

- [ ] Port `duration/iso8601_parser.rb` to
      `packages/activesupport/src/duration/iso8601-parser.ts`, including the
      `ParsingError` class and its message.
- [ ] Reduce `Duration.parse` to Rails' two lines: construct the parser, call
      `parseBang()`, then `new Duration(Duration.calculateTotalSeconds(parts), parts)`.
- [ ] Delete the `parse -> order:calculateTotalSeconds,constructor` row from
      the exclude shard by hand (only-shrink, no reseed;
      `parity:api:calls:tighten` for a stale mark).
- [ ] Existing `parse` tests keep passing (invalid inputs still raise).
