---
title: "Re-point two source comments citing landed stories at their open trackers"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 20
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Two source comments promise work against stories that have already landed, so
they read as pending debt when the tracker considers them finished:

- `packages/activerecord/src/associations/has-one-associations.test.ts:545-546`
  cites `has-one-loaded-target-create-missing-remove-target`
  (`status: done`, PR 4904).
- `packages/activerecord/src/relation-load-async.trails.test.ts:40` cites
  `install-executor-hooks-for-async-queries-tracker`
  (`status: done`, PR 6529).

Both were found by sweeping the `CONVERGEABLE (story ...)` citations during
PR #6850, which fixed the same defect on
`postgresql/schema-definitions.ts` after
`scripts/stale-story-references.test.ts` caught it there. These two are not
currently flagged by that guard (it does not read them as promises in these
positions), so they are silent.

Unlike the pg case, **the underlying deviations are still real** — this is a
re-pointing job, not a delete:

- The has-one one is already tracked by the open draft story
  `converge-has-one-remove-target-awaitable-arms` (0023-surfaced-deviations).
  Rails runs `remove_target!` inside `HasOneAssociation#replace`
  (`vendor/rails/activerecord/lib/active_record/associations/has_one_association.rb:69`);
  trails runs it from `_createRecord` because sync `replace`/`setNewRecord`
  cannot await.
- The async-queries one is tracked by
  `install-executor-hooks-async-queries-tracker-stopgap`
  (0023-surfaced-deviations).

## Acceptance criteria

- [ ] Each comment cites the open story that actually tracks its deviation, not the landed one.
- [ ] `pnpm vitest run scripts/stale-story-references.test.ts` stays green.
- [ ] No behavioural change; comments only.
