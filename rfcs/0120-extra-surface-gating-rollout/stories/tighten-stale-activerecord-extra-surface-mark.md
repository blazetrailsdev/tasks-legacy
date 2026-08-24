---
title: "Tighten activerecord's stale extra-surface mark to the measured novel/total"
status: draft
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: null
packages: []
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

`activerecord`'s committed extra-surface mark sits **above** the current
measurement, so the freeze is holding at a number the codebase has already
moved past. Measured on `main` @79f3e13d1 while fixing the RFC 0061 red:

```text
$ pnpm parity:api:extra:gate
extra-surface gate: activerecord novel mark 399 is above the current 391 —
  narrow it with `pnpm parity:api:extra:tighten`.
extra-surface gate: activerecord total mark 1424 is above the current 1411 —
  narrow it with `pnpm parity:api:extra:tighten`.
extra-surface gate: OK (arel novel 0/0, total 63/63;
  activerecord novel 391/399, total 1411/1424)
```

That is **8 novel / 13 total of unclaimed slack**. The gate exits 0 — a stale
mark is reported, not failed (by design) — so nothing forces the reconciliation
and the slack silently accumulates. Its practical cost is that 8 new novel
public names could land in `activerecord` today without turning the gate red,
which is exactly the growth #6997 enrolled the package to stop.

The mark was committed by #6997 (`scripts/api-compare/extra-surface-mark.json`)
at `{ "novel": 399, "total": 1424 }`; sibling PRs merging the same day removed
surface without tightening it.

## Converged shape

Run the sanctioned narrowing writer — the only writer for this file, and
only-shrink per dimension by construction:

```bash
pnpm parity:api:extra:tighten
```

so the mark reads the measured `{ "novel": 391, "total": 1411 }` (re-measure at
claim time; the burndown is active, so the true figures will be lower still).

Do **not** reseed and do **not** raise any dimension — there is deliberately no
reseed command for this file, for the same reason the call baselines forbid one.

## Notes / relationship to sibling stories

- `burn-down-and-enroll-activerecord` (this RFC) tightens the mark _as the
  burndown lands_; this story is the one-off reconciliation of slack that has
  **already** accrued, and does not wait on that burndown.
- `reconcile-activerecord-non-zero-gate-enrollment` (this RFC) settles the
  separate question of whether a `novel: 399` entry may sit in `GATED_PACKAGES`
  at all (freeze vs enroll). Independent of this: whichever tier
  `activerecord` ends in, its mark should match the measurement.
- `guard-extra-surface-mark-against-format-drift` (this RFC) would make a stale
  mark detectable mechanically rather than by reading gate output.

## Acceptance criteria

- `scripts/api-compare/extra-surface-mark.json` has `activerecord` at the
  measured novel/total, written by `pnpm parity:api:extra:tighten`.
- `pnpm parity:api:extra:gate` prints no "mark is above the current" line for
  `activerecord` and still exits 0.
- No dimension raised; no reseed; `arel`'s mark untouched.
