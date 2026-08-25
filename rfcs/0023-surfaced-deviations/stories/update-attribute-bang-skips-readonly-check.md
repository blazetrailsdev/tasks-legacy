---
title: "update-attribute-bang-skips-readonly-check"
status: draft
updated: 2026-07-31
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Found by the prism-codegen conformance scorer triage (PR #5727,
docs/infrastructure/prism-codegen-spike.md, "candidate untracked
deviations"). Rails update_attribute! calls verify_readonly_attribute(name)
before writing (vendor/rails/activerecord/lib/active_record/persistence.rb:552-558);
the port's updateAttributeBang (packages/activerecord/src/persistence.ts:1170-1177)
goes straight to writeAttribute + saveBang with no readonly check. Unless
save-time enforcement covers it, a readonly attribute raises in Rails and
silently writes in trails.

## Acceptance criteria

- A regression test that fails on baseline: update_attribute! on a
  readonly attribute must raise (read the Rails test for the exact error).
- Port calls verifyReadonlyAttribute (or equivalent) before the write.
