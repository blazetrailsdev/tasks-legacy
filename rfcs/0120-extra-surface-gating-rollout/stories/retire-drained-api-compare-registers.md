---
title: "Delete the three drained api-compare exclude registers"
status: ready
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: api-compare
packages: []
deps: []
deps-rfc: []
est-loc: 150
priority: 3
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Three api-compare exclude registers are fully drained and have been for some
time — all three files are literally `[]` (3 bytes):

- `scripts/api-compare/arity-exclude.json`
- `scripts/api-compare/inheritance-exclude.json`
- `scripts/api-compare/body-pins.json`

CLAUDE.md names `arity-exclude.json` and `inheritance-exclude.json` in its list
of deviation registers ("a burndown ledger, not a settled decision"). They have
been burned down. An empty register still costs: a loader, a validation path, a
schema, a test, a mention in the docs, and — worst — a visible invitation to add
a row to a tree nobody is watching, which is exactly the "never widen an
allowlist to cover new work" failure CLAUDE.md forbids.

This is a cleanup, not a migration. There is nothing to convert to JSDoc; the
RFC's JSON→JSDoc split classifies these as "already drained — delete".

## Acceptance criteria

- Confirm each file is `[]` on current `main` before deleting (a sibling PR may
  have added a row; if so, STOP and report rather than deleting the row).
- Delete the three JSON files plus their loaders, type definitions, validation,
  and tests.
- Remove the now-dead code paths in the consuming compare/lint scripts. A
  consumer that degrades to "always empty" must be removed, not left reading a
  missing file with a `?? []` fallback — that fallback is how a register comes
  back silently.
- Remove `arity-exclude.json` and `inheritance-exclude.json` from CLAUDE.md's
  register list, leaving the remaining registers named.
- `pnpm parity:api` and `pnpm parity:api:calls` pass unchanged; no delta in any
  reported number.
- If any of the three turns out to be load-bearing while empty (e.g. its absence
  changes a default), keep that one and say why in the PR body.
