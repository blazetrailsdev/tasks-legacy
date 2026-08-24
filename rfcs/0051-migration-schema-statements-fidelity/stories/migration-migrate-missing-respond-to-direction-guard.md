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

trails' `Migration#migrate`
(`packages/activerecord/src/migration.ts:1331-1344`, as of PR #6971) has no
such guard. It announces "migrating"/"reverting", checks a connection out of
`DatabaseTasks.migrationConnection().pool`, and calls `execMigration`
unconditionally.

### Corrected premise (2026-08-24, PR #6978)

This story was filed claiming a migration defining neither `up` nor `down`
returns early there, and that the check therefore has to distinguish a
subclass-provided `up`/`down` from an inherited base one. **Both claims are
wrong**, and the port must not implement them:

`ActiveRecord::Migration` defines instance `up` and `down` on itself
(`migration.rb:951-960` — the legacy-delegate shape, `self.class.delegate =
self` then `return unless self.class.respond_to?(direction)`). So
`respond_to?(:up)` and `respond_to?(:down)` are true for **every** migration
instance, change-only ones included, and the guard at `:965` gates nothing in
practice — a change-only migration runs in both directions, which is how
`exec_migration` (`migration.rb:985-998`) reaches its `respond_to?(:change)`
arm at all. Confirmed against MRI.

Resolving the guard against subclass-defined `up`/`down` would therefore skip
every change-based migration — a divergence from Rails, not a convergence.
`instance_method_already_implemented?` is not the analogue: it exists to stop a
GENERATED attribute reader shadowing a hand-written method, a question
`respond_to?` here never asks.

## Acceptance criteria

- [x] `Migration#migrate` opens with the port of `migration.rb:965`'s
      `return unless respond_to?(direction)`, before the announce / checkout /
      `execMigration` / `write` sequence.
- [x] The check is the literal `respond_to?` question — true for both
      directions on every migration, per "Corrected premise" above. The
      rationale lives in `migrate`'s JSDoc, at the call site.
- [x] A test pins that a change-only migration still runs in BOTH directions,
      so the subclass-only reading cannot be reintroduced later.
- [x] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
