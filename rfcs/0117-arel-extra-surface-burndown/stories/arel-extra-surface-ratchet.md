---
title: "Gate arel's extra-surface count with an only-shrink ratchet"
status: claimed
updated: 2026-08-22
rfc: "0117-arel-extra-surface-burndown"
cluster: null
packages: ["arel"]
deps: ["arel-root-and-barrel-tail"]
deps-rfc: []
est-loc: 180
priority: 11
pr: null
claim: "2026-08-22T23:57:28Z"
assignee: "wave-5f-head-sweep"
blocked-by: null
closed-reason: null
---

## Context

arel's extra surface grew every measured day from 2026-08-05 to 2026-08-22
(250 → 258 total, 86 → 89 novel; see this RFC's README for the CI-log
measurement). Nothing prevented it: unlike the call-set and call-argument
gates, `parity:api:extra` has **no baseline and no gate** — it prints a table
and exits 0.

Burning the population to zero without a ratchet just resets the clock. This
story lands the gate that keeps it there.

Existing prior art to mirror, not reinvent:

- `pnpm parity:api:calls` — the RFC 0047/0084 call-set ratchet: a committed
  per-file high-water mark, only-shrink, with `parity:api:calls:tighten` for
  narrowing a converged shard.
- `pnpm parity:api:calls:args` — the RFC 0095 twin over the same shards.
- `scripts/api-compare/extra-surface.ts` already emits `--json` with per-file
  `novelCount` / `movedCount`, which is the input a mark file needs.

**Important sequencing:** land this **after** the burndown stories, or at
least after `arel-operator-spellings-in-conventions` and the two `accept`
stories. Pinning a high-water mark at 258 would ratify the debt this RFC
exists to remove — CLAUDE.md is explicit that a baseline row is "we know this
is wrong and haven't fixed it yet", never a licence.

## Approach

- A per-package (not per-file, to start) committed mark for arel: novel count
  and total count, only-shrink.
- A CI check that fails on any increase, with the same
  "converged something? delete the row / tighten the mark" workflow as the
  call gates, and the same **no-reseed** rule.
- Scope the gate to arel only in this story. Extending it to activemodel and
  activerecord is a separate decision with a much larger population behind it
  — if it looks trivially generalizable, note that in the PR body but do not
  widen the gate here.

## Acceptance criteria

- A committed arel extra-surface mark and a gate script that fails on an
  increase in either novel or total, wired into the existing
  `Rails API/Test Comparison` CI job.
- A deliberate local experiment proves the gate is red on an added public
  name and green after removing it — a gate that has never failed is not
  known to work.
- The mark is set from the **post-burndown** measurement, and the PR body
  quotes the number it pins.
- Documented in CLAUDE.md's "Before you open the PR" list next to the other
  `parity:*` gates, with the only-shrink and no-reseed rules stated.
