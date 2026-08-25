---
title: "Converge Relation#to_xml onto activesupport's Array#to_xml"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
deps:
  - resolve-the-in-closure-xml-conversions
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

`Relation#toXml` (`packages/activerecord/src/relation/delegation.ts:1119`)
carries an inline re-implementation of `Array#to_xml`
(`activesupport/lib/active_support/core_ext/array/conversions.rb:183-211`).

In Rails there is no such body in ActiveRecord: `delegation.rb:101` is
`delegate :to_xml, ... to: :records`, and the implementation lives entirely in
the activesupport core-ext. trails has no `Array#to_xml` — the
activesupport array conversions live in the invented
`packages/activesupport/src/array-utils.ts` (`toSentence`, `toFs`,
`toFormattedS`) and `packages/activesupport/src/core-ext/array/conversions.ts`
does not exist at all, though `conversions.test.ts` beside it does.

PR #6813 rewrote that body as a faithful port of conversions.rb:183-211 (it
previously called a `Model#toXml` that had no Rails counterpart and has now been
deleted), so the correct shape is already written down — it is just sitting in
the wrong package, at the wrong seat, reachable only through a Relation.

## Converged shape

- `Array#to_xml` lands at its Rails seat in activesupport, beside the other
  `core_ext/array/conversions.rb` members.
- `activerecord/src/relation/delegation.ts` drops the inline body and delegates,
  matching `delegate :to_xml, to: :records` (`delegation.rb:101`) — the same
  shape `toSentence` / `toFs` / `toFormattedS` already use via
  `RECORD_DELEGATES`.
- Port the two clauses the inline version still omits:
  - `options[:indent] ||= 2` and `Builder::XmlMarkup.new(indent: options[:indent])`
    (`conversions.rb:187-188`). `IndentedXmlStringBuilder` hardcodes two spaces,
    so `indent:` is silently ignored. Rails pins this:
    `activesupport/test/core_ext/array/conversions_test.rb`
    `test_to_xml_with_indent_set` asserts `indent: 4`.
  - `yield builder if block_given?` (`conversions.rb:209`).

## Dependency

Needs `resolve-the-in-closure-xml-conversions`
(0098-activesupport-ar-closure-port), which owns the decision about where
`Array#to_xml` may live given XmlMini's out-of-closure classification. That
story covers the activesupport half; this one is the ActiveRecord-side consumer
flip and the two omitted clauses.

## Acceptance criteria

- `packages/activerecord/src/relation/delegation.ts` has no XML-building body;
  `to_xml` reaches the activesupport implementation.
- `indent:` is honored, pinned by a test named after
  `test_to_xml_with_indent_set`.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative;
  `pnpm parity:api:calls` and `:args` clean with no reseed.
