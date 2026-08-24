---
title: "Pilot review-then-migrate of call-mismatch rows on globalid and rack"
status: ready
updated: 2026-08-24
rfc: "0000-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 4
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/call-mismatches-exclude/**` is 201 shards, 558 rows, 832K —
by far the largest remaining side-channel register. Its rows are per-site facts
(`call-mismatch-baseline.ts:63-76`: `package + tsFile + rubyName + call +
reason`; 417 `kind: "calls"`, 141 `kind: "args"`), and their inline twin already
exists and is already load-bearing: `@missingRailsCall` / `@missingRailsArgs`
suppress the flag through the extractor
(`scripts/api-compare/missing-rails-call-tags.ts:8-12`).

**The constraint that shapes this story.**
`missing-rails-call-tags.ts:20-46` defines both seed strings
(`DEFAULT_REASON`, RFC 0047; `NARROW_DEFAULT_REASON`, RFC 0044) and states that
a tag carrying one is deliberately NOT a justification — `justifies()` rejects
it, "so if it suppressed, the whole baseline would be blessed by prose nobody
wrote". RFC 0083 already stopped `parity:api:build` from minting such a tag: it
"only writes a tag it has curated prose to migrate". Nearly every row in the
tree carries a seed string verbatim.

So a mechanical conversion is impossible by construction — it would produce
hundreds of non-suppressing tags and a red gate. The migration contract is
**review-then-migrate, per cluster**: a seeded row either CONVERGES (make the TS
body call what Rails calls; delete the row) or is REVIEWED into real per-entry
prose and migrated to a tag at the declaration. `parity:api:build` is the tool.

This story is the PILOT, on the two smallest trees, to establish the per-row cost
before the remaining clusters are chartered.

## Acceptance criteria

- Scope: `call-mismatches-exclude/globalid/**` and
  `call-mismatches-exclude/rack/**` only. Both are small (globalid's
  `identification.json` holds 2 rows; rack's `body-proxy.json` holds 1).
- Every row in scope is resolved as CONVERGED (row deleted, TS body now calls
  what Rails calls) or MIGRATED (real reviewed prose, tag at the declaration,
  row deleted). No row is left, and no row is bulk-moved carrying seed prose.
- Cite the Rails `file:line` behind each decision — `pnpm rails:find <query>` or
  `vendor/rails/` directly.
- The baseline is only-shrink: delete rows by hand, never `--write`/reseed
  (CLAUDE.md). Deleting rows lowers the source's unreviewed count below its
  committed high-water mark, so narrow it with
  `pnpm parity:api:calls:tighten <package>/<file>.json` — the named shards only.
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` both pass.
- The PR body reports the measured per-row cost (rows converged vs migrated, and
  roughly how long), because the remaining ~555 rows are chartered against that
  number.
