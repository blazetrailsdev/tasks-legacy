---
title: "Narrow activerecord's stale extra-surface marks (novel 399 to 391, total 1424 to 1411)"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "arel"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`pnpm parity:api:extra:gate` (RFC 0117's extra-surface ratchet,
`scripts/api-compare/lint-extra-surface-ratchet.ts`) passes but prints two
stale-mark warnings on every run:

```text
extra-surface gate: activerecord novel mark 399 is above the current 391 — narrow it with `pnpm parity:api:extra:tighten`.
extra-surface gate: activerecord total mark 1424 is above the current 1411 — narrow it with `pnpm parity:api:extra:tighten`.
extra-surface gate: OK (arel novel 0/0, total 63/63; activerecord novel 391/399, total 1411/1424)
```

Observed on every gate run across PR #7006 (before its diff, after it, and
after a rebase onto `main`), so it is not that PR's doing — sibling
convergence lowered activerecord's measured novel/total without anyone
narrowing the committed marks in
`scripts/api-compare/extra-surface-mark.json`.

The gate is only-shrink, so a mark sitting 8 novel / 13 total above the
measurement is dead headroom: new invented activerecord surface can land
under it without turning the gate red, which is exactly what the ratchet
exists to prevent.

## Converged shape

Run `pnpm parity:api:extra:tighten` (it writes each dimension DOWN and never
up) so activerecord's marks match the measurement, and commit the narrowed
`extra-surface-mark.json`. There is deliberately no reseed for this gate; do
not raise either number.

Confirm the numbers on a fresh `pnpm build` first — per
`project_api_extra_totals_move_with_build_state_not_commit`, `parity:api:extra`
totals track build state, not the commit, so a stale `dist/` reports a
different measurement.

## Acceptance criteria

- `pnpm parity:api:extra:gate` prints no stale-mark warning lines.
- `extra-surface-mark.json` changes only downward, and only for activerecord.
- No source change.
