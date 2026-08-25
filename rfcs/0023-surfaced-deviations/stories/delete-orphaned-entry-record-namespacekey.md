---
title: "Delete orphaned entry-record.ts namespaceKey helper"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 15
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

After PR #5105 converged `Store#namespaceKey`
(packages/activesupport/src/cache/store.ts:498) to Rails
`Cache::Store#namespace_key` semantics, the trails-only helper
`namespaceKey(key, namespace?)` in
packages/activesupport/src/cache/entry-record.ts:19-21 has zero callers
(grep: only its own definition remains). It is the pre-resolved-string
invention the wide-exclude used to point at; keeping it invites new callers
bypassing the per-call-override/callable semantics.

## Acceptance criteria

- entry-record.ts `namespaceKey` deleted (and any barrel export removed);
  typecheck and cache suites pass.
