---
title: "ParameterFilter#filter's no-filters arm spreads instead of params.dup, flattening HashWithIndifferentAccess"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ParameterFilter#filter`'s no-filters arm spreads into a plain object:

```ts
// packages/activesupport/src/parameter-filter.ts:181
return this.noFilters ? { ...params } : this.call(params);
```

Rails duplicates the receiver, preserving its class
(`vendor/rails/activesupport/lib/active_support/parameter_filter.rb:83-85`):

```ruby
def filter(params)
  @no_filters ? params.dup : call(params)
end
```

The `call` arm already gets this right — it allocates
`new (params.constructor ?? Object)()` to mirror `params.class.new`
(`parameter_filter.rb:126`, ported at `parameter-filter.ts:179-190`) — so a
`HashWithIndifferentAccess` survives a filtering pass but is flattened to a
plain object when the filter list is empty. That is exactly what
`test_parameter_filter_should_maintain_hash_with_indifferent_access`
(`vendor/rails/activesupport/test/parameter_filter_test.rb:88-99`) asserts, with
a table whose two rows are `["blah"]` and `[]` — the second row is the one that
takes the `no_filters` path, so the test cannot be ported until this converges.

## Converged shape

The no-filters arm duplicates through the receiver's own `dup` where it has one
(`HashWithIndifferentAccess#dup` exists at
`packages/activesupport/src/hash-with-indifferent-access.ts:360`), falling back
to the plain-object spread only for a bare object literal, which has no `#dup`
in JS. Keep it inline in `filter` — Rails has no helper here, so do not extract
one.

## Acceptance criteria

- `parameter_filter_test.rb` reports 0 assertion-count / 0 kind / 0 value in
  `pnpm parity:test -- --assertions --package activesupport`, with
  `assert_instance_of ActiveSupport::HashWithIndifferentAccess` passing for both
  table rows.
- `pnpm parity:api:extra --package activesupport` gains no novel surface.
