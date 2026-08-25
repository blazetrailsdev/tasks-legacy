---
title: "hash_filter must slice the params and extract permit_value"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "actionpack"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6665 (RFC 0108 foreign-receiver call marking), which stopped
the same-file closure from crediting `hash_filter` with an unrelated same-file
`slice` body and left the omission visible. Baselined there with a citation
(`call-mismatches-exclude/actioncontroller/metal/strong-parameters.json`,
`hash_filter` → `slice`); this story is the convergence that retires that row.

Rails (`actionpack/lib/action_controller/metal/strong_parameters.rb:1349-1359`):

```ruby
def hash_filter(params, filter, on_unpermitted: …, explicit_arrays: false)
  filter = filter.with_indifferent_access

  # Slicing filters out non-declared keys.
  slice(*filter.keys).each do |key, value|
    next unless value
    next unless has_key? key
    result = permit_value(value, filter[key], on_unpermitted:, explicit_arrays:)
    params[key] = result unless result.nil?
  end
end
```

trails (`packages/actionpack/src/action-controller/metal/strong-parameters.ts:692-760`,
`Parameters#_hashFilter`) is a wholesale rewrite: it iterates
`Object.entries(filter)`, guards each key with `k in this._data`, and inlines
the whole `permit_value` branch tree (`EMPTY_ARRAY` / `EMPTY_HASH` /
`array_filter?` / `explicit_arrays` / `non_scalar?`, `:1361-1380`) as a nested
`if/else` over `val instanceof Parameters` / `Array.isArray(val)` /
`isPlainObject(val)`.

The key set walked is the same, so this is not a behaviour bug — it is a
decomposition and control-flow divergence, and it is why the `permit_value` row
is baselined in the same shard.

## Converged shape

- `hashFilter` walks `this.slice(...Object.keys(filter))` and keeps Rails'
  three guards in Rails' order (`next unless value`, `next unless has_key? key`,
  `result.nil?`), rather than iterating the filter and testing `k in this._data`.
- Extract `permitValue(value, filter, { onUnpermitted, explicitArrays })` as its
  own method, mirroring `strong_parameters.rb:1361-1380` branch for branch, and
  call it from `hashFilter` — one Rails method is one TS method.
- Retire both `hash_filter → slice` and `hash_filter → permit_value` from
  `scripts/api-compare/call-mismatches-exclude/actioncontroller/metal/strong-parameters.json`
  by hand (only-shrink; do not reseed), then
  `pnpm parity:api:calls:tighten actioncontroller/metal/strong-parameters.json`.

## Acceptance criteria

- `hashFilter` calls `slice` and `permitValue`; both baseline rows are deleted.
- Existing strong-parameters tests stay green (no test renames — the port's
  behaviour must not change).
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
