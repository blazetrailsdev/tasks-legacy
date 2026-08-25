---
title: "Measure the arm-mismatch noise floor against the one-third tripwire"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-parity-tooling
packages: []
deps: ["promote-call-skeletons-to-an-arms-report"]
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

This RFC pre-commits, in its Open questions, to a tripwire: **if more than
roughly one third of reported arm mismatches are lowering artefacts rather than
real divergences, take the ungated path** and record the measured rate rather
than tuning a gate until it agrees with us.

That threshold was written down before any measurement, deliberately — writing
it first is what stops it being adjusted to fit the result. This story is the
measurement.

The prior art says the signal is real but unmeasured at this population. The
2026-08-05 codegen audit (referenced in `call-sequence-parity-in-wide-ratchet`,
RFC 0084) sampled 18 divergent rows from `relation.rb` and found roughly two
ninths were divergences **only** an ordered comparison catches — e.g.
`relation.rb::create`, where the generated skeleton is `ref:map ref:create` and
the port's is `loop ref:push ref:create ref:build ref:save`, because the port
hand-expanded `map` into a loop and inlined `create`. A call-set diff reports
nothing there.

But that was over prism-codegen's 10-file population. The skeleton artifact now
spans ~1,462 rows across 12 packages, and the noise profile at that scale is
unknown.

## Converged shape

A hand-audited random sample from `pnpm parity:api:arms:report`, each row
classified into exactly one of:

- **real** — a genuine missing / invented / reordered arm (this RFC's burndown);
- **lowering artefact** — the same control flow spelled differently across the
  language boundary (Ruby `unless` vs an inverted TS `if`, a `case` lowered to a
  lookup table or object literal, a guard clause hoisted above a block, a
  ternary vs an `if`, `&&`/`||` short-circuit vs a statement);
- **extraction bug** — the skeleton itself is wrong on one side.

Sample size large enough to separate ⅓ from ½ with confidence — 60-80 rows is
the usual band for that; state the number and the seed so the audit is
reproducible.

Note `foldSkeletonTokens` (`compare.ts:303`) already folds Ruby block-iteration
onto `loop`, so that specific artefact class should NOT appear. If it does, that
is an extraction bug and a finding in its own right.

## Acceptance criteria

- [ ] A reproducible sample (stated size + seed) drawn from the arms report,
      each row classified real / lowering artefact / extraction bug, with the
      per-row verdict recorded.
- [ ] The measured artefact rate is written into this RFC's Changelog **whatever
      it is** — a "too noisy to gate" result with a number attached closes this
      story successfully.
- [ ] Any extraction bugs found are filed as their own stories against RFC 0084
      (they are defects in its delivered tooling, not in this RFC's burndown).
- [ ] The verdict explicitly answers the tripwire: gate, or run ungated.

## Deps

Depends on the arms report existing.
