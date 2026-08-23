---
title: "Widen the extra-surface ratchet past arel, one burned-down package at a time"
status: draft
updated: 2026-08-23
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

PR #6897 (RFC 0117 `arel-extra-surface-ratchet`) landed the extra-surface
ratchet machinery and scoped it to arel **deliberately**, per that story: "if it
looks trivially generalizable, note that in the PR body but do not widen the
gate here." This story is the widening, and it supersedes the second half of
`wire-extra-surface-into-ci` (which asked for the gate to exist at all — it now
does).

What already exists on `main`:

- `scripts/api-compare/extra-surface-mark.json` — the committed marks, currently
  `{"arel": {"novel": 0, "total": 63}}`.
- `scripts/api-compare/extra-surface-mark.ts` — `GATED_PACKAGES`, `measure`,
  `exceedances`, `staleMarks`, `tightened` (a per-dimension `Math.min`, so a
  tighten can never launder a regression), `unmeasuredPackages`.
- `scripts/api-compare/lint-extra-surface-ratchet.ts` — the CLI, wired into the
  `Rails API/Test Comparison` job; `pnpm parity:api:extra:gate` /
  `pnpm parity:api:extra:tighten`.
- Documented in CLAUDE.md's "Before you open the PR" list with the only-shrink
  and no-reseed rules.

Widening is mechanically one line (`GATED_PACKAGES`), which is exactly why it
was held back: the populations behind the other packages are large and unburned.
Measured 2026-08-23 with `pnpm parity:api:extra`:

    package        novel  total
    activerecord     394   1424
    activesupport    295   1066
    actiondispatch   190    394
    trailties        141    267
    activemodel       82    223

Pinning those numbers as marks would ratify the debt — CLAUDE.md is explicit
that a baseline row says "we know this is wrong and haven't fixed it yet", never
a licence. So each package needs its own burndown decision before it is gated,
and this story is the decision plus the wiring, not a blind widening.

## Converged shape

Per package, in priority order (activemodel first — smallest population):
decide the burndown, land it, then add the package to `GATED_PACKAGES` with a
mark set from the POST-burndown measurement, quoting the number in the PR body.
Never add a package at its current number.

## Acceptance criteria

- [ ] At least one further package gated, with its mark set from a
      post-burndown measurement, not from today's total.
- [ ] The red/green experiment repeated for the newly gated package — a
      deliberately added public name turns the gate red, removing it turns it
      green.
- [ ] `GATED_PACKAGES` still excludes any package whose population has not been
      burned down; a package is never added at its current number.
- [ ] CLAUDE.md's gate description updated to name the gated set.
