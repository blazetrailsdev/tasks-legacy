---
title: "parity:api can replay a deleted worktree's run, even under API_COMPARE_FORCE=1"
status: draft
updated: 2026-08-24
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

`pnpm parity:api` / `pnpm parity:api:calls` can serve a whole run — manifests,
eslint sidecars, and the ratchet verdict — that was produced in a **different,
since-deleted worktree**. It is non-deterministic and is NOT prevented by
`API_COMPARE_FORCE=1`, which CLAUDE.md documents as the remedy for a warm cache.

Measured on PR #6964, same worktree, same branch, minutes apart:

| run                   | `Wrote .../eslint/rails-private-methods.json` path                               | ratchet line                    |
| --------------------- | -------------------------------------------------------------------------------- | ------------------------------- |
| plain                 | `worktrees/restore-relation-values-hash-75a9` (deleted)                          | 1488 baselined, 1154 unreviewed |
| `API_COMPARE_FORCE=1` | **own worktree**                                                                 | 417 baselined, 317 unreviewed   |
| `API_COMPARE_FORCE=1` | `worktrees/collection-proxy-delegate-query-method-bangs-to-scope-4106` (deleted) | 1487 baselined, 1158 unreviewed |

417/317 is this branch's true measurement (it matches `origin/main`'s state and
was reproduced on a run whose `Wrote` path was the own worktree). The 1487/1488
rows are some other branch's numbers entirely.

Two consequences, both bad:

- **A green ratchet verdict from a replayed run proves nothing about your
  branch.** An agent that reads `call-mismatches ratchet: OK` off a replay has
  verified nothing, and there is no error or warning — only the `Wrote ...` path
  betrays it.
- The replayed paths point at worktrees that no longer exist, so the artifacts
  the run claims to have written are not on disk where the next step looks for
  them. Downstream steps then fail with a confusing staleness error
  (`output/ts-api.json predates the api-compared package sources`).

Entry points: `scripts/api-compare/run.sh`, `scripts/api-compare/orchestrate.ts`,
and the shared cross-worktree cache the run reports as
"served from cache (N from the shared cross-worktree cache)".

Sibling stories in this family: `api-compare-shared-worktree-cache`,
`api-compare-shared-cache-eviction`, `api-compare-cache-key-resolved-read-set`,
`api-compare-resolution-shape-key-is-workspace-global`.

## Acceptance criteria

- A cache entry cannot supply an output path belonging to a different worktree.
  Either the cached payload stores paths relative to the run's own repo root and
  they are re-anchored on read, or the worktree root is part of the cache key
  for anything path-bearing.
- `API_COMPARE_FORCE=1` deterministically bypasses whatever layer is replaying;
  if a separate layer is responsible, it gets its own documented bypass.
- A run whose resolved output root is not the invoking worktree fails loudly
  rather than reporting `OK`, so a replay can never be mistaken for a verdict.
- Regression cover: a test that primes the shared cache from worktree A, then
  runs in worktree B, and asserts B's manifests land under B and carry B's
  measurements.
