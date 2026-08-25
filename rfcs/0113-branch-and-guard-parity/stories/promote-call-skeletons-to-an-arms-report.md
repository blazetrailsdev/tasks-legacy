---
title: "Decide the skeleton merge rule and ship parity:api:arms:report"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: arm-parity-tooling
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

This RFC's Rollout was written on the assumption that the arm/skeleton stream
had to be built. **It does not — it already ships**, delivered by RFC 0084:

- `scripts/api-compare/extract-ruby-api.rb:2119` — `collect_method_skeleton(body_node)`,
  attached as `method_info[:skeleton]` (`:684-685`, `:742-743`).
  Story `ordered-call-skeleton-stream-in-extractors`, PR #6161.
- `scripts/api-compare/extract-ts-api.ts:2662` — `extractSkeleton(node.body)`,
  emitting `if` / `loop` / `try` / `throw` plus normalized call and property
  reaches, over the real TypeScript compiler API. Same story/PR.
- `scripts/api-compare/compare.ts:303` — `foldSkeletonTokens`, which folds Ruby
  block-iteration onto `loop` so the two languages' spellings of the same
  construct compare equal. Story
  `fold-ruby-block-iteration-onto-loop-in-skeletons`, PR #6163.
- `scripts/api-compare/compare.ts:3905` — the paired artifact is written to
  `output/call-skeletons{mode}.json` for **every compared (Ruby, TS) pair**, not
  just flagged ones. Story `call-sequence-parity-in-wide-ratchet`, PR #6152.

`compare.ts:894-904` states the deliberate stopping point:

> Recorded for every compared pair, not just the flagged ones: this is the
> population call-sequence-parity seeds from, not a mismatch list. **Signal only
> (RFC 0084)** — written to its own output/call-skeletons.json, so
> call-mismatches.json and its ratchet read exactly what they read before, and
> **nothing gates on it yet**.

and names the missing piece exactly:

> Those are set operations by construction; **a sequence needs its own merge
> rule**, and until call-sequence-parity decides what it is, carrying the body's
> own stream and nothing else is the honest shape.

So the work here is not extraction. It is **deciding the merge rule and
surfacing the comparison**, over an artifact that already covers ~1,462 distinct
(package, file, method) rows across 12 packages — not the 10-file population the
retired prism-codegen scorer had (`retire-prism-codegen`, RFC 0086).

## Converged shape

A `parity:api:arms:report` script, mirroring the shape of
`parity:api:calls:args:report` (`scripts/api-compare/report-call-args.ts`,
wired in `package.json` as `pnpm tsx … --report`), that reads
`output/call-skeletons.json` and reports, per pair, a mismatch in **arm count**
or **arm order**.

The merge rule is the deliverable. Rails' own `catalog.ts:skeletonDiff` prior art
(described in `call-sequence-parity-in-wide-ratchet`) took the multiset
difference in both directions, which sees calls the TS body makes that Rails does
not. Pick and document one of:

- strict sequence equality (highest signal, highest noise);
- multiset equality + a separate `reordered` verdict (the retired scorer's
  `matched` / `reordered` split); or
- control-token subsequence only, ignoring the interleaved call reaches.

Do **not** compare predicate semantics — that boundary is stated in this RFC's
Non-goals and is what keeps this from becoming RFC 0108's problem.

## Acceptance criteria

- [ ] `pnpm parity:api:arms:report` exists, reads `output/call-skeletons.json`,
      and prints per-pair arm mismatches grouped by package/file.
- [ ] The chosen merge rule is documented at the call site with the reason the
      other two were rejected, and `compare.ts:894-904`'s "a sequence needs its
      own merge rule" comment is updated to name the decision.
- [ ] Report-only: nothing gates, no baseline is seeded, and
      `pnpm parity:api:calls` / `:args` read exactly what they read before.
- [ ] The report distinguishes arm-COUNT from arm-ORDER mismatches, since this
      RFC's clusters (`missing-arm`, `invented-arm`, `arm-order`) burn down
      differently.
