---
title: "nokogiri-backend-raises-syntax-error"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6920
claim: "2026-08-23T15:27:26Z"
assignee: "nokogiri-backend-raises-syntax-error"
blocked-by: null
closed-reason: null
---

# `XmlMini_Nokogiri#parse` raises a plain `Error` where Rails raises `Nokogiri::XML::SyntaxError`

## Context

Surfaced in PR #6839, which parameterized the XmlMini engine suite over every
backend and so had to name each backend's `expansion_attack_error`.

Rails (`activesupport/lib/active_support/xml_mini/nokogiri.rb:27`):

```ruby
raise doc.errors.first if doc.errors.length > 0
```

`doc.errors.first` is a `Nokogiri::XML::SyntaxError` _instance_, which is what
`nokogiri_engine_test.rb:12` and `hash_ext_test.rb:1026` both name as the
expected class.

trails (`packages/activesupport/src/xml-mini/nokogiri.ts:131`) raises
`new Error(doc.errors[0].message)`, because `@blazetrails/nokogiri` exposes its
parse errors as plain `{ message }` records with no `SyntaxError` class to
raise. The SAX sibling was converged in #6839 — `nokogirisax.ts` now raises
Ruby's `RuntimeError` for its two bare `raise` forms (`nokogirisax.rb:34,38`) —
so this is the last arm of that pair still divergent.

Two assertion sites carry the deviation, each with a cited call-site comment:

- `packages/activesupport/src/xml-mini/xml-mini-engine.test.ts` — the
  `NokogiriEngineTest` invocation's `expansionAttackError`.
- `packages/activesupport/src/core-ext/hash-ext.test.ts` — the
  `XmlMini_Nokogiri` arm of `expansion count is limited`.

## Acceptance criteria

- [ ] `@blazetrails/nokogiri` exposes a `SyntaxError` class mirroring
      `Nokogiri::XML::SyntaxError`, and `parseXml` surfaces document errors as
      instances of it.
- [ ] `xml-mini/nokogiri.ts` raises that error object directly, mirroring
      `raise doc.errors.first` rather than re-wrapping its message.
- [ ] Both assertion sites above name the Rails class, and their deviation
      comments are deleted.
