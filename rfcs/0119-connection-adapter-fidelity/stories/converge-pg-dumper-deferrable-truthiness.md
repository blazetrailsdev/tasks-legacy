---
title: "Gate dumped deferrable: on truthiness like Rails, not !== undefined"
status: ready
updated: 2026-07-27
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 30
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/postgresql/schema-dumper.ts:247`
and `:268` gate the dumped `deferrable:` option on `!== undefined`:

```ts
if (ec.deferrable !== undefined) opts.push(`deferrable: ${JSON.stringify(ec.deferrable)}`);
if (uc.deferrable !== undefined) opts.push(`deferrable: ${JSON.stringify(uc.deferrable)}`);
```

Rails gates on **truthiness**, so a non-deferrable constraint emits no
`deferrable:` at all:

- `vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_dumper.rb:51`
  — `parts << "deferrable: ..." if exclusion_constraint.deferrable`
- same file `:72` — `parts << "deferrable: ..." if unique_constraint.deferrable`

This became live when #5383 changed `extractConstraintDeferrable`
(`postgresql/schema-statements-class.ts`) to return `false` rather than
`undefined` for a non-deferrable constraint. That return value is correct — it
is what Rails' own tests assert (`migration/unique_constraint_test.rb:32`,
`migration/exclusion_constraint_test.rb:35`) — but it means the dumper's
`!== undefined` check is now always true and emits `deferrable: false` where
Rails emits nothing.

Rails pins the expected output in
`vendor/rails/activerecord/test/cases/schema_dumper_test.rb:250`:

```ruby
assert_match 't.unique_constraint ["position_1"], name: "test_unique_constraints_position_deferrable_false"', output
```

i.e. the non-deferrable line carries **no** `deferrable:` segment.

Not caught today because trails' `schema dumps unique constraints`
(`packages/activerecord/src/schema-dumper.test.ts:505`) asserts only with
`toContain` and never asserts the absence of `deferrable: false`.

Surfaced by review on #5384.

## Acceptance criteria

- Both checks in `postgresql/schema-dumper.ts` become truthiness checks, so a
  non-deferrable exclusion/unique constraint dumps no `deferrable:` option.
- `schema dumps unique constraints` / `schema dumps exclusion constraints` are
  strengthened to assert the full emitted line (Rails' `assert_match` strings),
  so the absence of `deferrable: false` is pinned rather than incidental. The
  strengthened assertion must fail on baseline.
- Test names stay verbatim; no renames.
