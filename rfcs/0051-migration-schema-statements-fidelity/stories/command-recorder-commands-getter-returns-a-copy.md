---
title: "CommandRecorder#commands returns a defensive copy where Rails' attr_accessor is live"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
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

`CommandRecorder#commands` is `attr_accessor :commands`
(`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:65`)
— it hands back the live array, which is what lets `change_table`'s bulk path do
`@commands << [...]` (`:142`) and `revert` splice it
(`:126-136`).

`packages/activerecord/src/migration/command-recorder.ts` exposes it as a getter
returning a defensive copy (`return [...this._commands]`), so a caller that
mutates what it gets back silently loses the write, and there is no `commands=`
writer at all where Ruby's `attr_accessor` gives one. Internal callers work only
because they touch the private `_commands` directly — which is itself the tell
that the public shape does not match.

Noticed while converging the command tuple to `[cmd, args, block]` in PR #7025
(RFC 0051 `command-recorder-tuple-has-no-block-seat`); out of scope there, and
the copy is load-bearing for nothing.

## Converged shape

`commands` reads and writes the live array, as `attr_accessor` does — a getter
returning `this._commands` plus the writer half (Rails' `commands=`), with the
internal `@commands <<` / splice sites reading through it the way the Ruby does.

## Acceptance criteria

- `commands` returns the live array, not a copy, and a writer exists matching
  Ruby's `attr_accessor :commands` (`command_recorder.rb:65`).
- `revert` and `changeTable`'s bulk path go through it as Rails does.
- `command-recorder{,.trails}.test.ts` and `invertible-migration.test.ts` green
  on SQLite, PostgreSQL and MySQL/MariaDB.
