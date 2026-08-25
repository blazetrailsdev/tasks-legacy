---
title: "Port Time.xmlschema to @blazetrails/date so XmlMini stops carrying its own copy of the stdlib regex"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "date"
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

`ActiveSupport::XmlMini::PARSING["datetime"]`
(`vendor/rails/activesupport/lib/active_support/xml_mini.rb:70`) is:

```ruby
"datetime" => Proc.new { |time| Time.xmlschema(time).utc rescue ::DateTime.parse(time).utc },
```

`Time.xmlschema` is Ruby stdlib (`ruby/lib/time.rb`), and
`@blazetrails/date`'s `Time` (`packages/date/src/time.ts`) carries the
**instance** `#xmlschema` for formatting (`time.ts:598`) but no `Time.xmlschema`
**reader**. #6465 therefore inlined that method's lexical-form regex into
`packages/activesupport/src/xml-mini.ts` (the `XMLSCHEMA` const) and gated
`Temporal.Instant.from` on it.

The gate is load-bearing and was measured, not guessed:
`Temporal.Instant.from` also accepts ISO **basic** format, so a bare
`Instant.from("2013-11-12T0211Z")` yields 02:11, whereas MRI's
`Time.xmlschema` raises on that string and Ruby falls through to
`DateTime.parse`, which reads no time at all and yields midnight. Verified
against the `ruby` on PATH; it is the assertion at
`vendor/rails/activesupport/test/xml_mini_test.rb:271`.

So the behaviour is right, but the regex lives in the wrong package: xml-mini
owns a copy of a stdlib parser, and any other caller needing `Time.xmlschema`
will either duplicate it or reach for the too-lenient `Instant.from`.

## Converged shape

A real `Time.xmlschema(str)` static (and its `Time.iso8601` alias — Ruby's
`lib/time.rb` aliases them, mirroring the existing instance-side
`Time.prototype.iso8601 = Time.prototype.xmlschema` at `time.ts:628`) on
`@blazetrails/date`'s `Time`, raising `ArgumentError`
(`invalid xmlschema format: <str>`) on a non-conforming lexical form, as MRI
does. Then `xml-mini.ts` deletes `XMLSCHEMA` and calls it, restoring the Ruby
body shape:

```ts
datetime: (time) => {
  try { return Time.xmlschema(String(time)).utc(); }
  catch { return DateTime.parse(String(time)) /* .utc */; }
},
```

Note the offset-less arm: `Time.xmlschema` treats a lexical form with no offset
as **local** time, where #6465's inline version reads it as UTC (documented at
that call site). Converging should either implement the local-time arm or
justify the UTC reading at the new call site.

## Acceptance criteria

- `Time.xmlschema` exists on `@blazetrails/date`'s `Time` with the MRI
  lexical-form acceptance and the MRI `ArgumentError` message; `Time.iso8601`
  aliases it.
- `xml-mini.ts` no longer carries its own `XMLSCHEMA` regex.
- `xml_mini_test.rb`'s `ParsingTest > datetime` still passes all five
  assertions, including the `"2013-11-12T0211Z"` basic-format case that the
  gate exists for.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
