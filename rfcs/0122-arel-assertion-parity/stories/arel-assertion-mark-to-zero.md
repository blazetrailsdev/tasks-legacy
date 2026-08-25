---
title: "Sweep the arel assertion residue and tighten the mark to zero"
status: claimed
updated: 2026-08-25
rfc: "0122-arel-assertion-parity"
cluster: null
packages: ["arel"]
deps:
  [
    "to-sql-visitor-assertion-parity",
    "select-manager-assertion-parity",
    "attribute-assertion-parity",
    "postgres-visitor-assertion-parity",
    "mysql-sqlite-visitor-assertion-parity",
    "dot-visitor-assertion-parity",
    "crud-manager-assertion-parity",
    "table-and-factory-assertion-parity",
    "nodes-cluster-assertion-parity",
  ]
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: "2026-08-25T16:18:38Z"
assignee: "collection-proxy-association-seat-is-degenerate-for-singular-names"
blocked-by: null
closed-reason: null
---

## Context

RFC 0122's closing story. After the per-file burndown stories land, arel's
three assertion dimensions should be at or near zero. This story sweeps
whatever residue the file-shaped stories did not own and tightens the mark to
`0 / 0 / 0`.

Two residues are known in advance:

- **512 extra (TS only) assertions.** `pnpm parity:test -- --assertions --package arel`
  reports these alongside the mismatch counts. Some are legitimate extra rigour
  and belong in a `.trails.test.ts` sibling; some are a single Rails assertion
  ported as three inside a mirrored test, which is drift and is also what drives
  much of the 178-strong `assertionCount` dimension.
- **The one-time `value`-mark correction.** RFC 0122's first story raises arel's
  `value` mark 17 → 79 because mapping `must_be_like` made 326 assertions
  value-comparable for the first time. That correction is only honest if the 79
  actually burn down; this story is where the RFC verifies they did.

Mark file: `scripts/test-compare/assertion-mismatch-mark.json`.
Ratchet: `scripts/test-compare/lint-assertion-mismatches.ts`.

## Acceptance criteria

- `pnpm parity:test -- --assertions --package arel` reports
  `0 assertion-count-mismatch, 0 assertion-kind-mismatch, 0 assertion-value-mismatch`.
- Any remaining trails-only assertion inside a mirrored test is either matched
  to its Rails counterpart or moved to a `.trails.test.ts` sibling. Nothing is
  deleted to make a number move.
- **No test name is renamed or reworded.**
- `assertion-mismatch-mark.json`'s arel entry reads
  `{ assertionCount: 0, kind: 0, value: 0 }`, and the four other gated packages'
  marks are no higher than they were before RFC 0122 opened
  (activemodel 307/**446**/59 after the mapping story, activerecord 1943/3926/37,
  activesupport 882/1227/104).
- `pnpm parity:test:assertions` is green.
- If a residue turns out to be a further tooling gap, it is fixed in
  `scripts/test-compare/assertion-kinds.ts` or the extractors with a
  justification citing both sides' semantics and an all-package before/after —
  never by loosening a mapping to make a real divergence disappear.
