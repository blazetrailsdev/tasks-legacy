---
title: "Burn down and enroll globalid"
status: draft
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: ["globalid"]
deps: ["extra-surface-mark-dimensions"]
deps-rfc: []
est-loc: 300
priority: 4
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Wave 1 of the enrollment rollout. From the 2026-08-24 run:

| package  | files | novel | moved | total | allowed | noCntrp |
| -------- | ----- | ----- | ----- | ----- | ------- | ------- |
| globalid | 4     | 5     | 11    | 16    | 5       | 13      |

The shape to notice: 5 novel but **13 no-counterpart** extras out of 16 total.
The work here is predominantly file MAPPING, not deletion — 13 of the extras are
in TS files no Rails file maps onto, scored with an empty allowed set
(`scripts/api-compare/extra-surface.ts:1439-1440`), so they are extras by
default rather than by invention. Check `PATH_SEGMENT_ALIASES` and
`RUBY_FILE_TS_OVERRIDES` in `scripts/parity/conventions.ts` before concluding
any name is invented; RFC 0117's §1 credited ~40 arel extras with a
conventions-only change and zero source change.

globalid also carries 5 `@noRailsEquivalent` tags already (2 by grep over
`packages/globalid/src` — the report's 5 counts inherited/interface coverage),
several inherited from RFC 0080's original migration out of
`extra-surface-allow.json`.

Use RFC 0117's triage model: delete / relocate to the Rails name / fold into the
ported method / tag LAST.

## Acceptance criteria

- `globalid` reaches `novel: 0` and `noCounterpartFiles: 0`.
- Enrolled in `GATED_PACKAGES` with a four-dimension mark per the RFC contract;
  `total` records whatever `moved` remains — `moved: 0` is NOT required.
- **Tag budget: globalid may finish with at most its current tag count.** A net
  new `@noRailsEquivalent` means the burndown relabelled rather than paid. A
  story needing one names the TypeScript language shortcoming in its PR body; a
  story needing two is blocked, not tagged.
- The PR body buckets every resolved name under the four triage headings by
  name, not by count (the RFC 0117 story convention).
- Any conventions.ts mapping fix is called out separately — it credits other
  packages too and is worth knowing about.
- `pnpm parity:api:extra:gate` passes.
