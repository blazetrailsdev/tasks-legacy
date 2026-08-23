---
title: "staleTagKey ignores the declaring class, so retiring one stale tag can delete a sibling's receipt"
status: in-progress
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: 6912
claim: "2026-08-23T12:57:31Z"
assignee: "converge-association-relation-inverse-wiring-onto-exec-queries"
blocked-by: null
closed-reason: null
---

## Context

`parity:api:build` retires a `@missingRailsCall` tag compare.ts reports STALE,
keyed by `staleTagKey(tsName, call)` (`scripts/api-compare/build.ts`) — the tag
site alone, with no owning class and no file. compare.ts's `staleCallTags`
(`scripts/api-compare/compare.ts`) records `{ tsFile, tsName, call }` for the
same reason: the tag rows it walks are keyed by file and name, not by the
class-qualified `callTagKey` the suppression side uses.

PR #6904 (this RFC, `call-set-migrator-cannot-tag-members-split-into-a-subdirectory`)
made the writer open the file the declaration actually lives in — a class trails
split into a subdirectory module, e.g. `cache.rb`'s `Store` in
`cache/store.ts` — by carrying `tsDeclFile` on the mismatch and suppression
rows. `staleTags` rows carry no such field, so the run passes ONE stale set to
every declaring file grouped under a row's `tsFile`.

That is safe for the migration itself, but it widens an existing key collision:
two declarations of the same `tsName` reachable from one row-file (a top-level
`foo` in `cache.ts` beside `Store#foo` in `cache/store.ts`, or two classes in
one file) share a `staleTagKey`, so retiring a stale tag on one can delete the
reviewed receipt on the other. The writer's whole preserve-don't-delete contract
(the comment above `expected` in `reconcileFileText`, added after PR #6873
dropped two receipts) exists to prevent exactly that class of loss.

## Converged shape

`staleCallTags` records the declaring class and file it already has in
`tsMissingCallTagsByFileName` / `declFileFor` — the `callTagKey(tsFile, tsClass,
tsName)` identity the suppression side is already keyed by — and `staleTagKey`
grows to match, so a stale tag is retired on the one declaration compare.ts
reported it for. `build.ts` then groups the stale set per declaring file instead
of passing the row-file's whole set to each.

No Rails counterpart: this is `scripts/api-compare/` parity tooling.

## Acceptance criteria

- [ ] `staleCallTags` rows carry the owning class (and declaring file where it
      differs), and `staleTagKey` is keyed on them.
- [ ] A tag on a same-named sibling declaration is NOT retired when compare.ts
      reports the other one stale — covered by a `build.test.ts` case that fails
      on the current key.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green; no
      baseline row added.
