---
title: "Test model registry is not an autoload: through-reflection resolution raises NameError for canonical models that exist"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Test model registry is not an autoload: through-reflection resolution raises NameError for canonical models that exist

## Context

Root cause of the SQLite CI red on PR #6723
(`through-reflection-source-name-swallows-nameerror`): once
`ThroughReflection#sourceReflectionName` stopped swallowing errors — as
`activerecord/lib/active_record/reflection.rb:1112-1130` requires, it has no
rescue — 10 tests across 6 files failed with
`NameError: Missing model class Contract` / `Categorization` for models that
exist in `packages/activerecord/src/test-helpers/models/`.

The path is entirely Rails-faithful on both sides:

`activerecord/lib/active_record/reflection.rb:758-780` — `automatic_inverse_of`
inverts `Developer#firm`, looks up `developers` on `Firm` (the plural lookup is
live because `activerecord/test/cases/helper.rb:40` sets
`automatically_invert_plural_associations = true`, which
`packages/activerecord/src/cases/helper.ts:48` mirrors), finds
`Company#developers` — a `through: :contracts` reflection — and calls
`valid_inverse_reflection?` → `reflection.foreign_key` → `source_reflection` →
`through_reflection.klass`.

Rails resolves `Contract` there because Zeitwerk autoloads the constant on
demand. trails registers a model only as a side effect of assigning an adapter
(`packages/activerecord/src/base.ts:1349`, `registerModelConstant` called from
the adapter setter), so a canonical model that no test has explicitly wired is
absent from the registry and `constantize` misses.

PR #6723 unblocked itself by registering `Contract` / `Categorization` in the 6
affected files, matching the idiom those files already use ("register the
canonical targets so `Developer`'s declared associations resolve"). That is a
per-test workaround for a systemic gap, and it scales badly: every new test that
touches a model whose associations transitively name an unregistered model reds
the same way, and the error names the _inner_ model, not the test's own subject.

Note a plain `import` does not fix it — importing `contract.js` evaluates the
class but registers nothing, because registration hangs off the adapter setter,
not the class definition. Verified during PR #6723 and reverted.

## Converged shape

Give model-constant resolution an autoload-shaped entry point so naming a
canonical model resolves it, the way Zeitwerk does for Rails:

- `constantize` (or the registry behind it) falls back to the canonical model
  index for `test-helpers/models/**` rather than requiring prior registration.
- Registration stops being a side effect of adapter assignment
  (`base.ts:1349`) — a class is resolvable because it is defined, not because
  something happened to give it an adapter.
- The per-test `registerModel(...)` calls added purely to satisfy transitive
  through-reflection resolution can then be removed.

`project_canonical_autoload_index_must_be_opt_in` is prior art on the index
being opt-in; this story is about the resolution fallback, not eager loading of
every model.

## Acceptance criteria

- [ ] Resolving a canonical test model by name succeeds without a prior
      `registerModel` call for that model.
- [ ] The `registerModel(Contract)` / `registerModel(Categorization)` calls PR
      #6723 added to `strict-loading.trails.test.ts`,
      `associations/inverse-associations.test.ts`, `relation/merging.test.ts`,
      `relation/or.test.ts`, `relation/where-chain.trails.test.ts` and
      `associations/has-one-through-disable-joins-associations.test.ts` are
      removed, and those suites stay green.
- [ ] No blanket `catch` is reintroduced anywhere in the through-reflection
      resolution path to paper over a miss.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
