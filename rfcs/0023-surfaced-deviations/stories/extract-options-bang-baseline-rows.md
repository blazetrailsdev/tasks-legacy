---
title: "extract_options! baseline rows: port Array#last and settle the receiver-arg call shape"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6454 added two call-mismatch baseline rows for
`Array#extract_options!` in
`scripts/api-compare/call-mismatches-exclude/activesupport/hash-utils.json`:

- `call: "last"` — Ruby's `last.is_a?(Hash)` (activesupport/lib/active_support/core_ext/array/extract_options.rb:23);
  the port reads `args[args.length - 1]`, an index expression with no call node.
- `call: "extractable_options?"`, `kind: "args"` — Ruby calls it on the receiver
  with no arguments (extract_options.rb:9); the port's
  `isExtractableOptions(last)` takes the receiver as its first argument because
  TS cannot reopen `Hash`.

Both are shape debt, not behavior debt, but they are rows in an only-shrink
baseline and should converge.

## Converged shape

A ported `Array`-class `last` (the `core-ext/array/access.ts` idiom, which
already holds `from`/`to`/`second`…) gives `extractOptionsBang` a real call to
make. The `args` row converges only if the receiver-as-first-parameter idiom
gains a gate-recognized form; if it does not, that is a whole-repo idiom
question and this story should be blocked pointing at it rather than reworded.

## Acceptance criteria

- The `last` row is deleted, with `extractOptionsBang` calling a ported `last`.
- The `extractable_options?` args row is deleted or the story is blocked with
  the idiom-level blocker named.
