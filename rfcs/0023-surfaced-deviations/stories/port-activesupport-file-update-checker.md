---
title: "ActiveSupport::FileUpdateChecker is unported; port file_update_checker.rb:35-163"
status: ready
updated: 2026-08-07
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
  - "trailties"
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

`ActiveSupport::FileUpdateChecker`
(`vendor/rails/activesupport/lib/active_support/file_update_checker.rb:35-163`)
has no trails port. `packages/activesupport/src` holds only
`evented-file-update-checker.test.ts`, whose five tests are all `it.skip` with
no source file behind them.

Rails treats it as the watcher API the framework is built on: `CheckPending`
takes it as an injected collaborator and instantiates it in `build_watcher`
(`vendor/rails/activerecord/lib/active_record/migration.rb:649, 675-682`),
`RoutesReloader` uses it, and the I18n reloader is the class's own documented
example (`file_update_checker.rb:25-34`). trails'
`packages/trailties/src/application/routes-reloader.ts` says so in its header
comment: "Rails' watcher half (FileUpdateChecker + Ruby `load`) ships with
autoloading later".

It is the blocker on [[check-pending-has-no-file-update-checker-watcher]]
(blocked), which needs a default for Rails'
`initialize(app, file_watcher: ActiveSupport::FileUpdateChecker)` and something
for `build_watcher` to construct.

Surfaced while shipping #6157.

## Converged shape

Port the four documented API methods (`file_update_checker.rb:8-34`):
`initialize(files, dirs = {}, &block)`, `updated?`, `execute`,
`execute_if_updated`, plus the private `watched` / `updated_at` / `max_mtime` /
`compile_glob` / `escape` / `compile_ext` (`:102-162`), keeping the mtime-in-
the-future skip (`:115-141`) and the "cached until the block is executed"
memo semantics of `@watched` / `@updated_at`.

**Decide the sync/async question first — it is the substance of this story.**
The Ruby API is synchronous over `File.mtime` / `Dir[]`, and this repo's hard
rule is async fs only. A `Promise`-returning `updated?` / `execute` is a
deviation every caller inherits, so pick the shape deliberately and justify it
at the call site rather than discovering it halfway through.

## Acceptance criteria

- `ActiveSupport::FileUpdateChecker` ported at the Rails file path and name,
  with the four public methods and the six private helpers, matching
  `file_update_checker.rb:35-163` method for method.
- The block-required `ArgumentError` (`:45-47`) is raised with Rails' message.
- Any sync→async deviation is justified at the call site as a language/repo
  constraint, per CLAUDE.md.
- Tests mirror `vendor/rails/activesupport/test/file_update_checker_shared_tests.rb`
  by name; the skipped `evented-file-update-checker.test.ts` stays untouched
  (that is the _evented_ subclass, a separate port).
