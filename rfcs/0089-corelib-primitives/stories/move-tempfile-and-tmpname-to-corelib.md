---
title: "Move the Tempfile / Dir::Tmpname port to the corelib package"
status: draft
updated: 2026-08-24
rfc: "0089-corelib-primitives"
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

PR #6984 ported Ruby stdlib `Tempfile` (`tempfile.rb`) and `Dir::Tmpname`
(`tmpdir.rb:114-163`) into `packages/activesupport/src/tempfile.ts`, following
this RFC's own precedent: unanchored Ruby primitives live in activesupport for
now and get re-homed together. It is tagged `@noRailsEquivalent CONVERGEABLE`
pointing here, alongside `range-ext.ts`, `include.ts`, `prepend.ts` and
`core-ext/string/succ.ts` (see `move-range-core-and-succ-to-corelib` and
`move-module-mixin-primitives`).

Three exported/module-level names move with it:

- `Tempfile` — `tempfile.rb`
- `createTmpname` / `random` / `UNUSABLE_CHARS` — `Dir::Tmpname.create`,
  `Dir::Tmpname::RANDOM.next`, `Dir::Tmpname::UNUSABLE_CHARS`
  (`tmpdir.rb:122`, `:126-136`, `:139-163`)
- `TempfileBasename` — the `String | [prefix, suffix]` type of the `basename`
  argument

Callers to update on the move: `packages/activesupport/src/encrypted-file.ts`,
`packages/activesupport/src/core-ext/file/atomic.ts`, and
`packages/activerecord/src/tasks/postgresql-database-tasks.ts`.

There is also a file-local `ensure()` helper in `tempfile.ts`, tagged
`@noRailsEquivalent CONVERGEABLE`. It spells Ruby's `begin ... ensure ... end`
for a block whose value is returned, branching on whether the block returned a
Promise, because `Tempfile.open`/`Tempfile.create` must serve both a synchronous
and an asynchronous block. It is duplicated logic that a corelib package could
host once for every primitive with the same shape.

## Acceptance criteria

- [ ] `Tempfile`, `Dir::Tmpname`'s three members and `TempfileBasename` live in
      the corelib package, at paths mirroring `tempfile.rb` / `tmpdir.rb`.
- [ ] The three call sites above import from the new home; the
      `@noRailsEquivalent CONVERGEABLE` tags that point at this RFC are deleted,
      not reworded.
- [ ] `ensure()` either moves to a shared corelib seat or is shown to be
      unnecessary there.
- [ ] `pnpm parity:api:extra --package activesupport` no longer reports
      `tempfile.ts`; `pnpm parity:api:calls` and `parity:api:calls:args` stay
      green.
