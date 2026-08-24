---
title: "stale-story-references whole-tree scans time out at the default 5s under host load"
status: in-progress
updated: 2026-08-24
rfc: "0061-ci-failures"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 6998
claim: "2026-08-24T18:07:08Z"
assignee: "stale-story-references-scan-times-out-under-load"
blocked-by: null
closed-reason: null
---

## Context

Surfaced in PR #6992 while confirming a CI red was unrelated to the diff.

`scripts/stale-story-references.test.ts` has two cases that scan the whole repo
tree and the tasks checkout:

- `no comment or markdown paragraph in the tree names a story that has already
landed` (`:146`) — `loadStories(resolveTasksDir(REPO_ROOT))` plus
  `scanStoryReferences(REPO_ROOT)`.
- `inventories stale citations in the frozen tree without gating on them`
  (`:152`) — `collectFrozenMarkdownFiles(REPO_ROOT)`.

Both run under vitest's default 5s `testTimeout`. On an unloaded host they take
~2.1s, comfortably inside it. On this shared dev box under sibling-agent load
(load average 32) they time out — and the failure is reported as
`Error: Test timed out in 5000ms`, which reads exactly like a hang rather than
contention. Re-running the same commit with `--testTimeout=60000` passes 17/17
in 4.68s of actual test time.

The cost is misdiagnosis: a full-tree scan timing out under load is
indistinguishable at a glance from the real thing this test exists to catch (a
genuinely stale story citation), and it fires on PRs whose diff cannot possibly
affect it. It also runs in the Unit Tests lane alongside 13 other
`scripts/test-compare` files, which is where it first tripped.

## Acceptance criteria

- The two whole-tree cases get a timeout sized to the work they actually do
  rather than inheriting the 5s default — an explicit per-test timeout is the
  minimal fix, and CI's own headroom is not an argument against it since the
  same suite runs locally on shared hosts.
- Preferred if cheap: cut the work instead of raising the ceiling — the tree
  scan and `loadStories` are pure reads and could be hoisted into a
  `beforeAll` shared by both cases rather than repeated per case.
- A timeout failure in these two cases is distinguishable from a real stale
  citation, e.g. the assertion message names the scan that overran.
- `pnpm vitest run scripts/stale-story-references.test.ts` passes on a loaded
  host (reproduce with a parallel `pnpm vitest run scripts/test-compare/`).
