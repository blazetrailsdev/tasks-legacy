---
title: "Enroll did-you-mean and actionpackversion in the extra-surface gate"
status: ready
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: ["extra-surface-mark-dimensions"]
deps-rfc: []
est-loc: 120
priority: 3
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two packages satisfy the RFC's enrollment contract today with no burndown at
all, from the 2026-08-24 comparison run:

| package           | files | novel | moved | total | allowed | noCntrp |
| ----------------- | ----- | ----- | ----- | ----- | ------- | ------- |
| did-you-mean      | 0     | 0     | 0     | 0     | 0       | 0       |
| actionpackversion | 1     | 1     | 2     | 3     | 0       | 3       |

`did-you-mean` is at a clean zero and is not enrolled — free work.
`actionpackversion` is one file with one novel name and three
no-counterpart extras, so it needs a look but not a campaign.

`GATED_PACKAGES` is `["arel"]` at
`scripts/api-compare/extra-surface-mark.ts:41`. `unmeasuredPackages()` (`:129`)
already fails the gate for a gated package a run never reported, which is the
guard that makes enrolling a zero-file package safe rather than vacuous.

Depends on `extra-surface-mark-dimensions` — the contract pins
`noCounterpartFiles` and `allowed`, so enrolling before those dimensions exist
would commit a two-dimension mark that has to be rewritten.

## Acceptance criteria

- `did-you-mean` enrolled at `{ novel: 0, total: 0, noCounterpartFiles: 0,
allowed: 0, internal: <measured> }`.
- `actionpackversion`: its 1 novel name and 3 no-counterpart extras are resolved
  by RFC 0117's triage model (delete / relocate / fold / tag — tag LAST), then
  enrolled at `novel: 0, noCounterpartFiles: 0`. Name the disposition of each in
  the PR body.
- If `actionpackversion` cannot reach `noCounterpartFiles: 0` because the file
  genuinely has no Rails counterpart, enroll `did-you-mean` alone and file the
  remainder as its own story rather than enrolling on a weaker contract.
- `pnpm parity:api:extra:gate` passes; the gate summary names all enrolled
  packages.
- Zero new `@noRailsEquivalent` tags. A package this small needing a tag is a
  signal to stop and ask, not to tag.
