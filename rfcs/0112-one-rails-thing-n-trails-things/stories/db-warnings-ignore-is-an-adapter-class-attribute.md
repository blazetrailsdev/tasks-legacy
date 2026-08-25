---
title: "db_warnings_ignore is an adapter class attribute, not a module config"
status: ready
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord.db_warnings_ignore` is a module-level config in Rails —
`singleton_class.attr_accessor :db_warnings_ignore` with `self.db_warnings_ignore = []`
(`vendor/rails/activerecord/lib/active_record.rb:259-263`) — and the one reader
is `AbstractAdapter#warning_ignored?`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract_adapter.rb:1227-1231`):

```ruby
def warning_ignored?(warning)
  ActiveRecord.db_warnings_ignore.any? do |warning_matcher|
    warning.message.match?(warning_matcher) || warning.code.to_s.match?(warning_matcher)
  end
end
```

trails holds it as a class attribute on the adapter instead —
`AbstractAdapter.dbWarningsIgnore` (`connection-adapters/abstract-adapter.ts`,
just above `isWarningIgnored`), read through
`(this.constructor as any).dbWarningsIgnore ?? []` at
`abstract-adapter.ts:2738`, and written by
`support/with-db-warnings-action.ts` and the PG cases in
`adapters/postgresql/postgresql-adapter.test.ts`.

This is the exact sibling of `db-warnings-action-resolved-per-adapter-not-once`
(PR #7055), which moved `db_warnings_action` from the same trails-only class
attribute onto the `ActiveRecord` namespace object in `ar-config.ts`. The
`ignore` half was left in place there to keep that PR scoped; it is the only
remaining `AbstractAdapter.dbWarnings*` class attribute.

A second, smaller divergence rides along in the same body: Rails matches with
`String#match?`, so a String matcher is a **regexp source**, not a substring —
trails uses `msg.includes(m)` for the String arm. Rails also stringifies a nil
code (`warning.code.to_s`) rather than skipping the code check when it is
absent, where trails guards on `warning.code !== undefined`.

## Converged shape

- `ActiveRecord.dbWarningsIgnore` on the `ar-config.ts` namespace object,
  defaulting to `[]` (`active_record.rb:262-263`), with
  `AbstractAdapter.dbWarningsIgnore` deleted.
- `isWarningIgnored` reads that namespace config and mirrors
  `abstract_adapter.rb:1228-1230` arm-for-arm: `match?` semantics for both the
  message and the `to_s`'d code, with no `undefined` guard Rails does not have.
- `support/with-db-warnings-action.ts` scopes the namespace config (it already
  scopes the action half), and the PG warning cases poke the same surface.

## Acceptance criteria

- [ ] No `AbstractAdapter.dbWarningsIgnore` reference remains in the tree.
- [ ] `isWarningIgnored` matches `abstract_adapter.rb:1227-1231` in both arms.
- [ ] `allowlist of warnings to ignore` / `allowlist of warning codes to ignore`
      (`adapters/postgresql/postgresql-adapter.test.ts`) stay green, as do
      mysql2's `warnings_test.rb` counterparts on the MySQL lane.
- [ ] `pnpm parity:api:calls` / `parity:api:extra --package activerecord` clean;
      no new baseline rows.
