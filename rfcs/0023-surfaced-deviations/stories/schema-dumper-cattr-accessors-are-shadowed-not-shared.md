---
title: "schema-dumper-cattr-accessors-are-shadowed-not-shared"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
  - "activesupport"
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

Rails declares the dumper's ignore/pattern filters with `cattr_accessor`
(`vendor/rails/activerecord/lib/active_record/schema_dumper.rb:17-41`:
`ignore_tables`, `fk_ignore_pattern`, `chk_ignore_pattern`,
`excl_ignore_pattern`, `unique_ignore_pattern`). A Ruby `cattr_accessor` is a
**single shared slot**: writing it through `ConnectionAdapters::SchemaDumper`
and reading it through `ActiveRecord::SchemaDumper` sees the same value.

trails ports them as plain TS `static` fields on the base class. JS static
inheritance is prototypal, so a write through the subclass creates an **own**
property that shadows the base's, and a read off the base misses it.

PR #6140 hit this: after `ConnectionAdapters::SchemaDumper` became a real
subclass, `migration/foreign-key.test.ts` set `SchemaDumper.fkIgnorePattern`
through the subclass while the dumper body read it off the base, and the
`addForeignKey` assertion failed. It was worked around by keeping that file (and
`support/schema-dumping-helper.ts`) importing the **base** for the static while
constructing through the subclass — which works but encodes a rule no Rails dev
would infer.

## Converged shape

The five accessors behave as one shared slot regardless of which class in the
hierarchy reads or writes them, the way `cattr_accessor` does — e.g. accessors
that delegate to a single module-level cell, as
`@blazetrails/activesupport`'s `classAttribute` / `cattrAccessor` support does
elsewhere. Call sites then stop caring whether they hold the base or the subclass.

## Acceptance criteria

- [ ] `ignoreTables`, `fkIgnorePattern`, `chkIgnorePattern`, `exclIgnorePattern`
      and `uniqueIgnorePattern` read and write one shared slot across
      `SchemaDumper` and `ConnectionAdapters::SchemaDumper`.
- [ ] `migration/foreign-key.test.ts` and `support/schema-dumping-helper.ts` can
      import either class and still work.
- [ ] A regression test writes through the subclass and reads through the base.
