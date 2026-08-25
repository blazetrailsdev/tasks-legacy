---
title: "Delete the duplicate bespoke json-encoding.test.ts"
status: draft
updated: 2026-08-03
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
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

`packages/activesupport/src/json-encoding.test.ts` and
`packages/activesupport/src/json/encoding.test.ts` both declare
`describe("TestJSONEncoding")` and both claim to port
`activesupport/test/json/encoding_test.rb`. The `json/encoding.test.ts` copy is
the one on the conventional path (`json/encoding.rb -> json/encoding.test.ts`);
the flat `json-encoding.test.ts` is a duplicate whose assertions are bespoke —
they call `JSON.stringify` directly instead of `ActiveSupportJSON.encode`, so
they exercise the JS built-in rather than the ported encoder and cannot catch
encoder regressions.

Surfaced in PR #5971: the same two `... with custom time precision` tests
existed in both files, in the flat copy asserting against `new Date(...)` +
`toISOString()` string surgery. Only the `json/encoding.test.ts` copy was made
faithful; the flat one still asserts nothing about our encoder.

## Converged shape

Delete `packages/activesupport/src/json-encoding.test.ts`, folding any test name
it covers that `json/encoding.test.ts` lacks into the latter as a faithful port
(routing through `ActiveSupportJSON.encode`, or `it.skip` with the Rails name if
the behaviour is not yet ported). One TS file per Rails test file, on the
conventional path.

## Acceptance criteria

- `json-encoding.test.ts` is gone; no duplicate `describe("TestJSONEncoding")`.
- Every Rails test name previously covered only by the flat file is present in
  `json/encoding.test.ts` (real or `it.skip`, verbatim Rails name).
- `pnpm parity:test` delta for `encoding_test.rb` is non-negative.
