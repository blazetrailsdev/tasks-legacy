---
title: "Migration#migrate drops Rails' respond_to?(direction) early return"
status: in-progress
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: 6978
claim: "2026-08-24T12:03:42Z"
assignee: "extra-surface-allow-reopened-module-method-files"
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::Migration#migrate`
(`vendor/rails/activerecord/lib/active_record/migration.rb:964-966`) opens with
an early return:

```ruby
def migrate(direction)
  return unless respond_to?(direction)
  ...
```

A migration that defines neither `up` nor `down` (only `change`, or nothing at
all) is a no-op in that direction: Rails returns before announcing, before
checking a connection out, and before `write`.

trails' `Migration#migrate`
(`packages/activerecord/src/migration.ts:1331-1344`, as of PR #6971) has no
such guard. It announces "migrating"/"reverting", checks a connection out of
`DatabaseTasks.migrationConnection().pool`, and calls `execMigration`
unconditionally — so a direction the migration does not answer still prints
both banners and performs a pool checkout Rails never performs.

The trails base class defines `up()`/`down()` on the prototype, so the JS
spelling of Rails' `respond_to?(direction)` is not a bare `typeof
this[direction] === "function"` — it has to distinguish a subclass-provided
`up`/`down` from the inherited base no-op, the same distinction
`instance_method_already_implemented?` draws elsewhere in the repo.

## Acceptance criteria

- [ ] `Migration#migrate` returns early when the migration does not answer
      `direction`, before the announce / checkout / `execMigration` /`write`
      sequence, mirroring `migration.rb:965`.
- [ ] The check resolves against subclass-defined `up`/`down` rather than the
      inherited base-class prototype methods.
- [ ] A test covers a migration answering only one direction: the unanswered
      direction announces nothing and performs no checkout.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
