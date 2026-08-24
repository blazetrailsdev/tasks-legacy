---
title: "stale-story-references gates on the tasks checkout, which no trails path filter can see"
status: draft
updated: 2026-08-24
rfc: "0025-fidelity-verification-tooling"
cluster: null
packages: []
deps: []
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

`scripts/stale-story-references.test.ts` compares citations found in the
trails tree against story frontmatter read from the **tasks repo checkout**
(`loadStories(resolveTasksDir(REPO_ROOT))`,
`scripts/stale-story-references.test.ts:147`). That second input lives in a
different repository, so no path filter over a trails diff can gate on it.

PR #6994 widened `UNIT_TESTS_PKGS_RE` to `packages/`, `scripts/` and `eslint/` —
the guard's `SOURCE_ROOTS` (`scripts/stale-story-references.ts:31`) — which
fixes the half of the problem that lives in trails: a citation added or
invalidated under `packages/activerecord/` now runs the lane. It cannot fix
the other half. Flipping a story to `status: done` in the tasks repo changes
this test's result with **no trails commit at all**, so the red first appears
on whatever unrelated push happens to land next.

That is exactly how the #6994 red reached `main`: #6980 closed
`converge-column-encode-with-init-with`, and the failure surfaced a push later
against #6989's unrelated activemodel diff, which is a badly misleading
signal — the bisect points at an innocent commit.

## Acceptance criteria

- A story transitioning to `done`/`closed` in the tasks repo cannot silently
  arm a future red in trails. Options to weigh, not a prescribed design:
  - the tasks CLI's `done`/`close` path greps the trails tree for citations
    of the slug and refuses (or warns loudly) while one is still pending;
  - trails CI runs the guard on a schedule against the tasks repo `main`, so
    the red is attributed to the story transition rather than to the next
    unrelated push;
  - the guard reports the offending story's landing PR in its failure message
    so the misattribution is at least legible to whoever picks it up.
- Whichever lands, the failure names the story transition that caused it, not
  just the citing `file:line`.
