---
title: "Nokogiri SyntaxError#level is a string union, not the gem's Integer with level predicates"
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

PR #6920 added `packages/nokogiri/src/xml/syntax-error.ts`, mirroring
`Nokogiri::XML::SyntaxError` (nokogiri gem `lib/nokogiri/xml/syntax_error.rb`),
so `XmlMini_Nokogiri#parse` can `raise doc.errors.first`
(`activesupport/lib/active_support/xml_mini/nokogiri.rb:27`).

Two shape divergences from the gem remain, inherited from the `XmlError`
record the class replaced:

- `#level` is a string union `"warning" | "error" | "fatal"`. The gem's
  `level` is an Integer, and the class exposes the predicates `none?`,
  `warning?`, `error?`, `fatal?` (`syntax_error.rb`), which trails does not.
- `XmlDocument.parse` (`packages/nokogiri/src/xml/document.ts:23-31`) only ever
  builds `"fatal"` errors, from `XmlParseError#details`. libxml2 warnings and
  recoverable errors never reach `#errors`, so a document that parses with
  warnings reports none, where the gem collects them.

## Converged shape

- `SyntaxError#level` is a number; `none()`, `warning()`, `error()`, `fatal()`
  compare against 0/1/2/3, mirroring `syntax_error.rb`.
- `XmlDocument.parse` surfaces libxml2's non-fatal diagnostics with their real
  level, not a hardcoded `"fatal"`.
- `packages/nokogiri/src/xml/document.test.ts:21`'s `expect(doc.errors[0].level)`
  moves with it.

## Acceptance criteria

- [ ] `level` is the gem's Integer, with the four predicates.
- [ ] Non-fatal libxml2 diagnostics appear in `XmlDocument#errors`.
- [ ] `xml-mini/nokogiri.ts` and both XmlMini test suites still pass unchanged.
