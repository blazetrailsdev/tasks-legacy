---
title: "parity:api:extra credits symbol-keyed [included] / [extended] hooks"
status: draft
updated: 2026-08-24
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
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

`parity:api:extra` counts a symbol-keyed Ruby lifecycle hook as **novel extra
surface**, even though it mirrors a real Ruby module method:

    activemodel  callbacks.ts   — 1 novel:  [extended]
    activemodel  attributes.ts  — 1 novel:  [included]

Both are the sanctioned trails spelling of Ruby's `self.extended` /
`self.included` (CLAUDE.md "Module mixins"; the symbols are exported from
`packages/activesupport/src/include.ts:122`), and both have exact Ruby
counterparts — e.g. `ActiveModel::Callbacks.extended`
(`activemodel/lib/active_model/callbacks.rb:66-70`).

CLAUDE.md already states the intent: "Because they are symbol-keyed they are not
public string-named members, so they never collide with the `SKIP_GROUPS` entry"
— but the extra-surface extractor does not implement that, so every correctly
ported hook shows up as invented surface. The row is noise in an ungated package
today; it would block the gate the day a gated package (arel) grows one, and it
mis-signals the shape as a deviation to anyone reading the report.

Surfaced by PR #6987, which added the `[extended]` hook to
`packages/activemodel/src/callbacks.ts`; `attributes.ts`'s `[included]`
predates it.

## Converged shape

The extractor recognizes a member keyed by
`Symbol.for("@blazetrails/activesupport:included")` /
`…:extended` and credits it against the Ruby module's `self.included` /
`self.extended`, rather than emitting a novel row. A hook with no Ruby
counterpart on the mapped module still reports as novel.

## Acceptance criteria

- [ ] `pnpm parity:api:extra --package activemodel` reports 0 novel for
      `callbacks.ts` and `attributes.ts`.
- [ ] A symbol-keyed hook whose Ruby module has no `self.included` /
      `self.extended` is still reported.
- [ ] `pnpm parity:api:extra:gate` unchanged for arel (0 novel / 63 total).
- [ ] A test in `scripts/api-compare/` pins both arms.
