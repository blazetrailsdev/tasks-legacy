---
title: "converge-test-fixtures-class-attribute-stores"
status: blocked
updated: 2026-08-25
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
pr: null
claim: "2026-08-25T17:14:37Z"
assignee: "converge-test-fixtures-class-attribute-stores"
blocked-by: "Premise does not hold: none of test_fixtures.rb:31-38's eight class_attributes exist in trails. packages/activerecord/src/test-fixtures.ts is the bespoke fixtures() DSL, not a port of ActiveRecord::TestFixtures, and a repo-wide grep finds no fixtureTableNames / fixtureClassNames / fixtureSets / useTransactionalTests / preLoadedFixtures / lockThreads / fixturePaths store, nor any hasOwnProperty copy-on-first-write helper for them. There is nothing to converge; declaring the eight would be new surface (a port of the module), which is a different story."
closed-reason: null
---

# Converge test_fixtures' 8 class_attribute stores onto classAttribute()

## Context

Split out of `converge-remaining-activerecord-copy-on-write-stores-onto-class-attribute`
(RFC 0112), which converged the other bucket-B clusters but ran out of LOC
budget before this one.

`vendor/rails/activerecord/lib/active_record/test_fixtures.rb:31-38` declares
eight class_attributes in one `included do` block:

```ruby
class_attribute :fixture_path, instance_writer: false
class_attribute :fixture_table_names, default: []
class_attribute :fixture_class_names, default: {}
class_attribute :use_transactional_tests, default: true
class_attribute :use_instantiated_fixtures, default: false
class_attribute :pre_loaded_fixtures, default: false
class_attribute :lock_threads, default: true
class_attribute :fixture_sets, default: {}
```

Audit each against `packages/activerecord/src/test-fixtures.ts`: the
container-valued ones (`fixture_table_names`, `fixture_class_names`,
`fixture_sets`) are the ones that carry no guard today, so an in-place `.push` /
`[k] =` silently mutates the parent's container. The scalar ones are already
correct as plain statics (bucket A of the parent story) and need no change —
but converge the containers, and cite the Rails line for anything left as a
plain static.

## Acceptance criteria

- [ ] Each container-valued declaration becomes a `classAttribute()` call with
      Rails' option set, or carries a call-site citation of the Rails construct
      it actually mirrors.
- [ ] No `hasOwnProperty` copy-on-first-write helper survives for any of the
      eight.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative;
      `pnpm parity:api:calls` and `:args` clean, no reseed.
