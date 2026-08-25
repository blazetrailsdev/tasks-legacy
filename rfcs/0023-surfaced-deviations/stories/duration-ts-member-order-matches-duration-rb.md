---
title: "duration.ts member order drifts from duration.rb (asJson, coerce)"
status: draft
updated: 2026-08-18
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

`packages/activesupport/src/duration.ts` does not follow
`vendor/rails/activesupport/lib/active_support/duration.rb`'s member order.
Surfaced while placing `Duration#sum` in PR #6685 (the reviewer caught the
commit message overclaiming): `sum` now sits immediately after `_parts`
(`duration.rb:481-486`) and before `raiseTypeError` (`:520`), which is right,
but the two members between them in the port are misplaced relative to Rails:

- `asJson` — `duration.rb:459-461`, right after `ago` (`:444-448`), well BEFORE
  `_parts`; in the port it trails `sum`.
- `coerce` — `duration.rb:245-254`, near the top of the class; in the port it
  trails `sum` too.

Nothing enforces this: `blazetrails/rails-file-structure-method-order` is
scoped to `arel` and `activemodel`
(`eslint/`-manifest consumers; see CLAUDE.md "Before you open the PR" step 4),
so activesupport member order drifts silently.

## Converged shape

- `duration.ts` members appear in `duration.rb` order — in particular `coerce`
  back near the arithmetic block (`:245`) and `asJson` back after `ago`
  (`:459`), leaving `_parts` (`:481`), `sum` (`:486`) and `raiseTypeError`
  (`:520`) contiguous as Rails has them.
- Body changes are none: this is placement only, verifiable with
  `git diff -M`.

## Acceptance criteria

- [ ] `duration.ts` member order matches `duration.rb`.
- [ ] No body edits — only moves.
- [ ] `parity:api` delta non-negative; `parity:api:calls` / `:args` green.
