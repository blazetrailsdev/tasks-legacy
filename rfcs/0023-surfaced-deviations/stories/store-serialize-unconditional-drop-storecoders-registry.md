---
title: "store() guards serialize and hoists the coder to feed a trails-only _storeCoders registry"
status: draft
updated: 2026-08-22
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

## Context

Surfaced by PR #6844, which converged `store -> serialize` in
`packages/activerecord/src/store.ts` but had to leave a
`@missingRailsArgs serialize — CONVERGEABLE` tag on `store` and a control-flow
divergence beside it. Both trace to one trails-only seam.

Rails (`activerecord/lib/active_record/store.rb:106-110`):

```ruby
def store(store_attribute, options = {})
  coder = build_column_serializer(store_attribute, options[:coder], Object, options[:yaml])
  serialize store_attribute, coder: IndifferentCoder.new(store_attribute, coder)
  store_accessor(store_attribute, options[:accessors], **options.slice(:prefix, :suffix)) if options.has_key? :accessors
end
```

Three lines, and the `serialize` call is UNCONDITIONAL: installing
`IndifferentCoder` as the attribute's type is how Rails makes the store column
deserialize to a `HashWithIndifferentAccess`, and it is what
`store_accessor_for` (store.rb:212-218) later reads back via
`type_for_attribute(store_attribute).accessor`.

trails diverges twice, both because of `_storeCoders`, a per-class
`WeakMap<typeof Base, Map<string, IndifferentCoder>>` declared in `store.ts`
with `setStoreCoder` / `getStoreCoder`:

1. **The coder is hoisted into a local** (`indifferentCoder`) because the same
   instance goes to both `setStoreCoder` and `serialize`, where Rails writes
   `IndifferentCoder.new(store_attribute, coder)` inline as the kwarg value.
   That is the `@missingRailsArgs` row.
2. **The `serialize` call is wrapped in a guard** —
   `if (!colType || typeof colType.accessor !== "function")` — that skips it for
   structured column types (json/jsonb/hstore). Rails has no such branch.

`storeAccessorFor` (store.ts, `ActiveRecord::Store#store_accessor_for`) then
reads the registry as a fallback after trying `typeForAttribute(...).accessor()`,
which is the only reason the registry has to exist: the type-side lookup does
not reliably answer for a text-backed store column, so the coder is remembered
out-of-band.

Note the whole file also carries a module-private `serialize` slot injected by
`base.ts` via `registerSerializeFn`, to break a `store -> serialize -> json ->
store` module cycle. That seam is fine and is NOT what this story converges —
it is already bound to the Rails name.

## Converged shape

Make `typeForAttribute(storeAttribute).accessor()` answer for every store
column, text-backed included, by calling `serialize` unconditionally so the
`Serialized` type carrying `IndifferentCoder` is always installed. Then:

```ts
serialize(modelClass, storeAttribute, {
  coder: new IndifferentCoder(storeAttribute, coder as CoderLike | null),
});
```

with `setStoreCoder` / `getStoreCoder` / `_storeCoders` deleted, and
`storeAccessorFor` reduced to Rails' two lines — `type_for_attribute(...)`, the
`respond_to?(:accessor)` guard raising `ConfigurationError`, then `.accessor`.
The `@missingRailsArgs` tag on `store` goes with it.

## Acceptance criteria

- [ ] `store` calls `serialize` unconditionally with the coder constructed
      inline, matching store.rb:108.
- [ ] `_storeCoders`, `setStoreCoder`, `getStoreCoder` are deleted.
- [ ] `storeAccessorFor` matches store.rb:212-218 with no registry fallback.
- [ ] The `@missingRailsArgs serialize` tag is removed; `pnpm parity:api:calls`
      and `pnpm parity:api:calls:args` green with no new baseline row.
- [ ] `pnpm parity:api:extra --package activerecord` loses the corresponding
      novel names.
- [ ] `store.test.ts` green on SQLite, PostgreSQL and MySQL/MariaDB — the
      structured-column arms (json/jsonb/hstore) are the ones the removed guard
      protected.
