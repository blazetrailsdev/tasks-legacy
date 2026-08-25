---
title: "ActiveSupport::JSON.encode throws on a bigint — jsonify short-circuits past Numeric#as_json"
status: draft
updated: 2026-08-20
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::JSON.encode` still throws on a JS `bigint`, because
`JSONGemEncoder#jsonify` never reaches `Numeric#as_json` for one.

`packages/activesupport/src/json/encoding.ts:78-100` — `jsonify`'s first arm
returns `string | bigint | null | true | false` unchanged, so a `bigint` is
handed straight to `stringify` (`:117`), which is `JSON.stringify` and throws
`TypeError: Do not know how to serialize a BigInt`.

PR #6787 gave `Numeric#as_json`
(`packages/activesupport/src/core-ext/object/json.ts`, Ruby
`activesupport/lib/active_support/core_ext/object/json.rb:110-114`) the decimal
-digits-as-string answer that keeps a bigint encodable, but `jsonify` bypasses
the dispatcher for exactly this value, so `ActiveSupportJSON.encode(1n)` is
still a throw while `asJson(1n)` is `"1"`.

Rails has no such split: `jsonify`'s first arm is
`when String, Integer, Symbol, nil, true, false` (encoding.rb:89-92) and Ruby's
own `Integer#to_json` renders the digits as a JSON _number_, which is why the
arm can short-circuit. JS has no encodable arbitrary-precision number, so the
short-circuit is what breaks.

## Acceptance criteria

- `ActiveSupportJSON.encode(1n)` and `encode({ id: 99999999999999999999n })`
  return valid JSON rather than throwing.
- The fix lands in `jsonify` at the arm that short-circuits (encoding.rb:89-92),
  routing a `bigint` to `Numeric#as_json` instead of past it — not by
  re-implementing the coercion in `stringify`.
- The deviation from Rails' JSON-number rendering is justified at the call site
  with the `json.rb:110-114` cite and the `@noRailsEquivalent PERMANENT`
  classification the extra-surface gate requires.
- A test in `packages/activesupport/src/json/` covers encode-of-bigint at both
  the scalar root and nested in a hash.
