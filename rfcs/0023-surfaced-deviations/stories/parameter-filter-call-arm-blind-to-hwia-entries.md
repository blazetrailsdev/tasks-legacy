---
title: "ParameterFilter#call traverses own properties, so a HashWithIndifferentAccess filters to empty"
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

`parameter-filter-no-filters-arm-drops-hash-class` states that "the `call` arm
already gets this right". It does not, and that premise will leave
`test_parameter_filter_should_maintain_hash_with_indifferent_access` failing
even after the `no_filters` arm is converged. Surfaced while scoping the
activesupport assertion-parity tail (PR #6641).

Rails' `call` walks the receiver through the Hash protocol and writes back
through `[]=` (`vendor/rails/activesupport/lib/active_support/parameter_filter.rb:125-133`):

```ruby
def call(params, full_parent_key = nil, original_params = params)
  filtered_params = params.class.new

  params.each do |key, value|
    filtered_params[key] = value_for_key(key, value, full_parent_key, original_params)
  end

  filtered_params
end
```

Ours allocates the right class but then reads and writes _own enumerable
properties_ (`packages/activesupport/src/parameter-filter.ts:187-196`):

```ts
const filteredParams = new ((params.constructor ?? Object) as ObjectConstructor)() as Record<
  string,
  unknown
>;

for (const [key, value] of Object.entries(params)) {
  filteredParams[key] = this.valueForKey(key, value, fullParentKey, originalParams);
}
```

`HashWithIndifferentAccess` keeps its entries in a private `Map`
(`packages/activesupport/src/hash-with-indifferent-access.ts:62`, `data`), not in
own properties. So `Object.entries(hwia)` yields `[]` and the index write lands
on the instance instead of the map: filtering a HWIA returns an **empty** HWIA
rather than a filtered one. The allocation is Rails-correct; the traversal is not.

## Converged shape

`call` must traverse and write through the receiver's own Hash protocol, so a
HWIA round-trips its entries — mirroring Ruby's `params.each` / `filtered_params[key]=`.
`HashWithIndifferentAccess` already exposes `entries()` (`:291`) and a setter; a
plain object keeps the `Object.entries` path. Both arms of
`parameter_filter_test.rb:88-99` (filter list `["blah"]` and `[]`) must return a
`HashWithIndifferentAccess` with its contents intact.

## Acceptance criteria

- Filtering a `HashWithIndifferentAccess` returns a `HashWithIndifferentAccess`
  with every key present and the matched values masked.
- `test_parameter_filter_should_maintain_hash_with_indifferent_access` ports as
  Rails writes it (`assert_instance_of`, both table rows).
- No new extra surface (`pnpm parity:api:extra --package activesupport`).
