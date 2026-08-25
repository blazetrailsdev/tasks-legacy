---
title: "call-gate-population-includes-test-helpers"
status: draft
updated: 2026-08-25
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
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

The call-set gate (`pnpm parity:api:calls`, RFC 0047/0084) decides whether a
Ruby callee name is "in population" by whether that name resolves anywhere on
the TS side. It also reads `packages/<pkg>/src/test-helpers/**`, which are test
support files with no Rails counterpart.

Observed in PR #7015: adding `packages/arel/src/test-helpers/uniq.ts` — a test
helper standing in for Ruby core `Array#uniq`, the sibling of the existing
`must-be-like.ts` — made the gate report a NEW mismatch in an unrelated SOURCE
file it had never flagged:

```text
+ arel  nodes/bound-sql-literal.ts  initialize  uniq
```

`bound_sql_literal.rb:20-21` does call `.uniq` twice, and
`packages/arel/src/nodes/bound-sql-literal.ts` expresses that dedupe as
`[...new Set(...)]`. Nothing about that file changed; only the resolvability of
the name `uniq` did. The mismatch had to be silenced with a
`@missingRailsCall uniq — PERMANENT` receipt at the constructor.

That is a false positive with a real cost: an unrelated PR is forced to add a
permanent deviation receipt to a source file, and the gate's population becomes
a function of what test helpers happen to exist.

## Converged shape

Exclude `src/test-helpers/**` from the name-resolution population the call-set
and call-argument gates build, the way `src/support/**` already sits outside
both compare populations. A test helper should never make a source file's
call set flag or unflag.

Then retire the receipt this forced: drop the `@missingRailsCall uniq` tag from
`packages/arel/src/nodes/bound-sql-literal.ts` and confirm
`pnpm parity:api:calls` stays green without it.

## Acceptance criteria

- `packages/*/src/test-helpers/**` no longer contributes callee names to the
  call-set / call-argument gate populations.
- The `@missingRailsCall uniq` receipt in `nodes/bound-sql-literal.ts` is
  removed and the gate is still green.
- No baseline row is added; per-file marks that tighten as a result are
  narrowed with `pnpm parity:api:calls:tighten`, never reseeded.
- Report the effect on every package's counts in the PR body.
