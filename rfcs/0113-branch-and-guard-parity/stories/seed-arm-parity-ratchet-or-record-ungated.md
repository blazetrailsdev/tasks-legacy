---
title: "Seed the arm-parity ratchet, or record the ungated decision"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-parity-tooling
packages: []
deps: ["measure-arm-mismatch-noise-floor"]
deps-rfc: []
est-loc: 250
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Conditional on the noise-floor measurement. This story has **two mutually
exclusive outcomes**, and both are a successful close — which is the point of
the tripwire this RFC committed to in its Open questions.

**If the artefact rate is acceptable:** promote `output/call-skeletons.json`
from signal to gate, on the only-shrink contract RFC 0084 and RFC 0095 already
established. Seed a baseline of the pre-existing mismatches, gate new ones, and
retire rows as this RFC's 59 burndown stories land.

**If it is not:** record the measured rate, delete nothing, and run the 59
stories ungated. Say so in the RFC changelog and close this story. Do **not**
seed a baseline that has to be widened to stay green — CLAUDE.md is explicit
that a register is a burndown ledger, and a gate whose baseline grows is
inverting the mechanism it exists to serve.

## Converged shape (gated path only)

Follow the established shape rather than inventing one:

- Baseline under `scripts/api-compare/` alongside `call-mismatches-exclude/`,
  sharded per package/file the same way, so `parity:api:calls:tighten`'s
  high-water-mark pattern applies unchanged.
- Rows carry a reviewed one-line `reason`; the seeded placeholder is never left
  in place for a row you add.
- Only-shrink: converging a divergence makes its row stale and reds the gate;
  the fix is deleting that one row by hand via `serializeBaseline` (sorted, not
  appended — an appended row passes the lint and reds CI's reseed-drift check),
  never `--write`/reseed.
- A `pnpm parity:api:arms` gating script beside `parity:api:arms:report`.

## Non-goals

- Gating `rescue` / `catch` arms in the first cut. This RFC's Open questions
  defer them: Rails' `translate_exception` family makes them load-bearing, but
  TS `try/catch` shapes differ more than `if/else` does. Include them in the
  report, exclude them from the gate until there is data.
- Comparing predicate semantics — stated in this RFC's Non-goals.

## Acceptance criteria

- [ ] Exactly one outcome is delivered and the other is explicitly ruled out in
      the RFC changelog, with the measured number that decided it.
- [ ] **Gated path:** `pnpm parity:api:arms` gates on an only-shrink baseline;
      every seeded row has a reviewed reason; the reseed-drift check is green;
      `parity:api:calls` / `:args` are unaffected.
- [ ] **Ungated path:** no baseline is added, the report stays wired and
      runnable, and this RFC's Rollout is edited to drop its gating phases so no
      later reader re-derives the decision.
- [ ] Either way, this RFC's Verification section reflects what was actually
      shipped.

## Deps

Depends on the noise-floor measurement.
