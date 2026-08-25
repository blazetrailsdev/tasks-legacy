---
title: "Duration::ISO8601Parser::ParsingError extends Error, not the ArgumentError port"
status: draft
updated: 2026-08-23
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

Rails: `activesupport/lib/active_support/duration/iso8601_parser.rb:13` —
`class ParsingError < ::ArgumentError; end`.

trails (PR #6909): `packages/activesupport/src/duration/iso8601-parser.ts:45-63`
spells it `export class ParsingError extends Error` with `override name =
"ParsingError"`. A caller cannot `catch`/narrow it as an ArgumentError, so
`Duration.parse` raises something no `instanceof ArgumentError` guard sees,
where Ruby's does.

Two independent blockers, both documented at the call site:

1. `eslint/rails-error-parity.mjs:50,157-165` — `ArgumentError` is in
   `ROOT_BASES`, and the rule requires the TS mirror of a root-based Rails
   error to extend a member of `NATIVE_ERRORS` (`Error`, `TypeError`, …).
   Extending `hash-utils.ts`'s `ArgumentError` port is a lint ERROR today:
   "Rails root error class `ParsingError` must extend a global Error type but
   extends `ArgumentError`".
2. Import cycle: `hash-utils.ts:6` -> `xml-mini.ts:7` -> `duration.ts:17` ->
   `duration/iso8601-parser.ts`. Importing the port closes it and evaluates the
   `extends` clause with its base in TDZ — measured on the PR branch as
   `TypeError: Class extends value undefined` while collecting
   `packages/activesupport/src/core-ext/duration.test.ts`.

Note `two-argumenterror-classes-break-instanceof` (0023, closed) is the
neighbouring problem: this package has ~10 local `class ArgumentError extends
Error` copies, so "the" ArgumentError is already ambiguous.

## Converged shape

`ParsingError extends ArgumentError`, with `ArgumentError` the single Ruby
`::ArgumentError` port, so `Duration.parse`'s raise narrows the way Ruby's
does. Getting there needs both blockers gone:

- Move the canonical `ArgumentError` out of `hash-utils.ts` into a leaf module
  with no runtime imports (the `ruby-empty.ts` shape), so nothing that raises it
  can close a cycle back through `xml-mini.ts`/`duration.ts`. `hash-utils.ts`
  re-exports it for existing callers.
- Teach `rails-error-parity` that a TS class whose manifest parent is a
  `ROOT_BASES` name may extend the trails PORT of that Ruby base, not only a
  `NATIVE_ERRORS` constructor — i.e. resolve the port by name rather than
  rejecting it. The rule's intent (a root must be Error-shaped) is satisfied
  transitively, since the port itself extends `Error`.

## Acceptance criteria

- [ ] `duration/iso8601-parser.ts`'s `ParsingError` extends the Ruby
      `ArgumentError` port; `Duration.parse("")` is `instanceof ArgumentError`.
- [ ] The justification block at `iso8601-parser.ts:45` is DELETED, not
      reworded.
- [ ] `packages/activesupport/src/core-ext/duration.test.ts` still green, and a
      plain-node import of the BUILT `dist/duration.js` as entry module does not
      throw (the TDZ check a vitest run masks).
- [ ] `pnpm lint`, `pnpm parity:api:calls`, `:args`, `:extra` green.
