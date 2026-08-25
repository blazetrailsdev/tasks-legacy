---
title: "config_accessor is private in Rails; trails' configAccessor is publicly callable"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Rails makes `config_accessor` a private class method:

```ruby
# vendor/rails/activesupport/lib/active_support/configurable.rb:128
private :config_accessor
```

so it is callable only from inside a class body (`config_accessor :foo`), never
as `SomeClass.config_accessor(:foo)`. `configurable_test.rb:123-129` asserts
exactly that:

```ruby
test "the config_accessor method should not be publicly callable" do
  assert_raises NoMethodError do
    Class.new { include ActiveSupport::Configurable }.config_accessor :foo
  end
end
```

trails' `Configurable.ClassMethods.configAccessor`
(`packages/activesupport/src/configurable.ts`) is a plain exported function
assigned onto the including class, so it is public. PR #6654 ported the file
and left that test as `it.skip("the config_accessor method should not be
publicly callable")` in `packages/activesupport/src/configurable.test.ts` — the
only skipped test in the file, and the reason the file is 9/10 rather than
10/10.

This matters beyond the test: every call site in the repo currently reaches
`configAccessor` through the public name, so nothing enforces the
declaration-time-only contract Rails guarantees.

## Converged shape

`eslint/rails-private-methods.json` is the existing mechanism for Ruby-private
members (built by `pnpm parity:api`; 604 files / 5002 names today) — check
whether `config_accessor` is already in it and whether the lint rule fires on
a `Klass.configAccessor(...)` call outside a class body. If the manifest is the
right home, the fix is to make sure the private marker is picked up from
`private :config_accessor` (the standalone `private :sym` form, as opposed to a
`private` section) and that the test can assert the violation.

Un-skip `the config_accessor method should not be publicly callable` in
`configurable.test.ts` — do not rename it.

## Acceptance criteria

- `the config_accessor method should not be publicly callable` runs (not
  skipped) and fails on the pre-fix baseline.
- `pnpm parity:test --package activesupport` shows configurable_test.rb at
  10/10 with 0 skipped, and its assertion count/kind/value stay 0/0/0.
- `pnpm parity:api` delta non-negative.
