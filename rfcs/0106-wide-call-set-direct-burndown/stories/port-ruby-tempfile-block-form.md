---
title: "Port Ruby's Tempfile block form and retire the two PERMANENT missingRailsCall receipts"
status: ready
updated: 2026-08-24
rfc: "0106-wide-call-set-direct-burndown"
cluster: null
packages: ["activesupport", "activerecord"]
deps: []
deps-rfc: []
est-loc: 260
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Ruby's `Tempfile` block form has no trails port, and three call sites have each
worked around it separately. Two of them carry a `@missingRailsCall ...
PERMANENT` receipt that names the gap explicitly:

- `packages/activesupport/src/encrypted-file.ts:144-148` —
  `@missingRailsCall create — PERMANENT`, for
  `encrypted_file.rb:90` `Tempfile.create(["", "-" + ...]) do |tmp_file|`. The
  body spells it as `fs.mkdtemp` plus an explicit `finally` unlink/rmdir
  (`encrypted-file.ts:161-...`), and the receipt says outright: "Ruby's Tempfile
  is stdlib, not Rails, and has no port".
- `packages/activerecord/src/tasks/postgresql-database-tasks.ts:213-218` —
  `@missingRailsCall open — PERMANENT` (Verified per-site, RFC 0106), for
  `postgresql_database_tasks.rb:132` `Tempfile.open("uncommented_structure.sql")`.
  Same workaround: "the fs port has no Tempfile analogue, so the body makes a
  temp dir with `mkdtempSync` and writes into it."
- `packages/activesupport/src/core-ext/file/atomic.ts:16-22` narrates a third
  stand-in — its yielded object "stands in for the `Tempfile` the Ruby block
  writes to" — porting `atomic.rb:24` `Tempfile.open(...) do |temp_file|`.

(`packages/rack/src/multipart/uploaded-file.ts:103` has a fourth, bespoke
`makeTempfile()` shim, but that one models Rack's uploaded-file object rather
than Ruby's `Tempfile`, so it is probably NOT in scope. Confirm while porting.)

Both PERMANENT receipts are mis-classified, and that is the point of this story.
Per CLAUDE.md, "a documented deviation is debt, not permission" — PERMANENT
means no TypeScript spelling can exist, and that is not true here. Nothing about
JS prevents a block-scoped temp file; the reason each site diverged is that
nobody had written the shared one. Once it exists, both receipts become
CONVERGEABLE and then deleted.

### Rails usage, for scope

`Tempfile` appears in 28 vendored files. The forms Rails actually uses:

- `Tempfile.create(basename) do |f| ... end` — creates, yields, then closes AND
  unlinks on block exit; returns the block's value.
- `Tempfile.open(basename) do |f| ... end` — yields, closes on block exit;
  removal is left to the finalizer rather than being immediate.
- The non-block form (`Tempfile.new`, `Tempfile.open` without a block), used by
  `activesupport/testing/stream.rb:25` and ActiveStorage.

**`basename` takes a String OR a `[prefix, suffix]` Array**, and the array arm
is load-bearing: `encrypted_file.rb:90` passes `["", "-" + content_path.basename...]`
precisely to control the suffix. Dropping the string arm or the array arm is
the classic silent gap (CLAUDE.md, "Symbols vs strings" has the same shape).

### Siting — open question, flag it in the PR

`Tempfile` is Ruby **stdlib** (a default gem), so it is neither Rails surface
nor an interpreter primitive. It therefore has no obviously correct package
today:

- RFC `0088-date-gem-port` established the "port the gem wholesale as its own
  package" model, but that is scaled to `date`; a whole package for one class is
  probably wrong.
- RFC `0089-corelib-primitives` is the eventual home for exactly this kind of
  unanchored emulation — but it is `status: postponed`, and it already lists
  `activesupport/src/range-ext.ts`, `include.ts`, `prepend.ts` and
  `core-ext/string/succ.ts` as unanchored primitives _living in activesupport
  for now_.

**Recommendation: follow that existing precedent** — put it in `activesupport`
beside the other unanchored primitives, tagged `@noRailsEquivalent`, and let
0089 re-home it along with the rest when it reactivates. Do not block this story
on 0089. If you disagree after reading both RFCs, raise it rather than silently
choosing differently.

Filesystem access must go through `packages/activesupport/src/fs-adapter.ts`,
which already exposes `mkdtemp` / `mkdtempSync` / `rmdir` (`fs-adapter.ts:96-115`)
because `EncryptedFile#change` needed them — see the existing `@noRailsEquivalent`
conventions there rather than reaching for `node:fs` directly.

### Async

The call sites are mixed: `encrypted-file.ts` is async, `postgresql-database-tasks.ts`
is async, `atomic.ts` is sync. The block form must serve both. Type the callback
as `(f) => T | Promise<T>` and have the wrapper return `T | Promise<T>` rather
than marking it `async` — an `async` wrapper defers scalar writes past
synchronous readers even when the body never truly suspends.

## Acceptance criteria

- A `Tempfile` port with `create` and `open`, each supporting the block form
  (yield, then close; `create` also unlinks) and returning the block's value,
  plus the non-block form. Both accept a String basename AND a
  `[prefix, suffix]` Array basename, and an optional `tmpdir`.
- Cleanup happens on the throwing path too, not just the normal return.
- All filesystem access goes through the activesupport fs adapter.
- `encrypted-file.ts` and `postgresql-database-tasks.ts` are converted to it,
  and **both `@missingRailsCall ... PERMANENT` receipts are DELETED**, not
  reworded — that is how this story is verified as done.
- `atomic.ts` either uses it or the PR states why its stand-in is a better
  match for `atomic.rb:24`.
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green; if
  retiring the receipts makes a baseline row stale, delete that one row by hand
  and run `pnpm parity:api:calls:tighten <shard>` — do not reseed.
- Tests cover: block value pass-through, unlink-on-normal-exit, cleanup-on-throw,
  the `[prefix, suffix]` arm, and the sync and async callback arms.

## Notes

`Tempfile` is stdlib, so there is no `vendor/rails` anchor for its behaviour —
check the semantics against real Ruby. `ruby` is on PATH; run it rather than
deriving the behaviour from memory (`Tempfile.create` unlinking on block exit
while `Tempfile.open` does not is exactly the kind of detail worth confirming
directly).
