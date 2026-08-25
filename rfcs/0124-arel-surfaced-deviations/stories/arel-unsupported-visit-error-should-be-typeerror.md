---
title: "Converge arel's UnsupportedVisitError onto Rails' plain TypeError"
status: in-progress
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: 7054
claim: "2026-08-25T16:56:36Z"
assignee: "arel-star-is-a-shared-const-not-a-per-call-method"
blocked-by: null
closed-reason: null
---

## Context

`packages/arel/src/errors.ts` declares an `UnsupportedVisitError extends
ArelError` that Rails does not have, and its own JSDoc ratifies the deviation
on preference grounds:

> Rails raises a plain `TypeError("Cannot visit #{class}")` from
> `Arel::Visitors::Visitor#visit`. Trails throws this named subclass of
> `ArelError` instead — same condition, but a named error class is more
> idiomatic in TS (callers catch by `instanceof`) […]

"more idiomatic in TS" is a preference, not a TypeScript language shortcoming,
so under CLAUDE.md's "converge, never ratify" rule this is debt, not a settled
decision. It surfaced while retiring the sibling `NotImplementedError` from the
same file in #6892 (RFC 0117 `arel-root-and-barrel-tail`), which converged the
identical shape the other way: Ruby's builtin `NotImplementedError` is now a
file-local class in `visitors/to-sql.ts` where Rails raises it, and it is gone
from `errors.ts` and from the `Visitors` barrel.

### Rails source

- `vendor/rails/activerecord/lib/arel/visitors/visitor.rb:14-19` — `def visit(object)` …
  `raise ::TypeError, "Cannot visit #{object.class}"` (the `unsupported` branch).
- `vendor/rails/activerecord/lib/arel/errors.rb` — declares `ArelError`,
  `EmptyJoinError`, `BindError`. Nothing else. There is no
  `UnsupportedVisitError` anywhere in the gem.
- `vendor/rails/activerecord/lib/arel/visitors/to_sql.rb:5` has no such class
  either, despite the trails JSDoc citing it as the Rails home.

### Where it is used today

- declared `packages/arel/src/errors.ts`
- re-exported from `packages/arel/src/visitors/to-sql.ts` and
  `packages/arel/src/visitors/index.ts` (public as `Visitors.UnsupportedVisitError`)
- thrown from `packages/arel/src/visitors/visitor.ts`
- pinned by `packages/arel/src/visitors/to-sql.test.ts`
  ("unsupported input should raise UnsupportedVisitError")

## Converged shape

Throw Ruby's condition with Ruby's class and Ruby's message from Ruby's raise
site — a `TypeError` carrying `Cannot visit <class>`, built from
`rubyClassName(object)`, raised in `visitor.ts#visit` and mirroring
`visitor.rb:18`. Delete `UnsupportedVisitError` from `errors.ts` and from both
barrel re-exports.

The pinning test keeps its Rails-matched name; only its assertion changes, from
`instanceof UnsupportedVisitError` to the `TypeError` + `"Cannot visit …"`
message, the way #6892 re-pinned the three `NotImplementedError` cases on their
Rails message strings.

Check whether any `activerecord` caller catches `UnsupportedVisitError` by
identity before removing the export; if one does, it should catch `TypeError`
plus the message, exactly as a Rails `rescue ::TypeError` would.

## Acceptance criteria

- `UnsupportedVisitError` gone from `packages/arel/src/errors.ts`,
  `visitors/to-sql.ts` and `visitors/index.ts`; `errors.ts` declares exactly the
  three classes `arel/errors.rb` declares.
- `visitor.ts#visit` raises `TypeError` with Rails' `"Cannot visit #{class}"`
  message, from the same guard and in the same branch order as `visitor.rb:14-19`.
- No new `@noRailsEquivalent` tag and no new baseline row.
- `pnpm parity:api:extra --package arel` stays at 0 novel.
- `pnpm vitest run packages/arel` green.
