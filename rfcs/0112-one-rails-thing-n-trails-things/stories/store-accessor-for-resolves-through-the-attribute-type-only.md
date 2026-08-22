---
title: "store_accessor_for resolves through the attribute type only, not a side coder registry"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 150
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `store_accessor_for` onto the record in PR #6427
(`call-args-ar-host-param-core`).

Rails `activerecord/lib/active_record/store.rb:219-225`:

```ruby
def store_accessor_for(store_attribute)
  type_for_attribute(store_attribute).tap do |type|
    unless type.respond_to?(:accessor)
      raise ConfigurationError, "the column '#{store_attribute}' has not been configured as a store. ..."
    end
  end.accessor
end
```

One lookup, one guard: the attribute's TYPE either responds to `accessor` or the
call raises. trails
`packages/activerecord/src/store.ts` `storeAccessorFor` keeps the type lookup but
adds a second resolution path — `getStoreCoder(modelClass, storeAttribute)`, a
per-class `WeakMap` registry `store()` populates (`setStoreCoder`) — and only
raises when BOTH miss. The registry exists because trails' `store()` resolves the
`IndifferentCoder` separately instead of handing it to `serialize` as the
attribute's type, the way `store.rb:106-109` does:

```ruby
serialize store_attribute, coder: IndifferentCoder.new(store_attribute, coder)
```

So the extra branch is a symptom: the coder never becomes the attribute's type,
so `type_for_attribute(...).accessor` cannot find it. `setStoreCoder`'s own JSDoc
already names itself a trails-only seam with no Rails counterpart.

## Converged shape

Have `store()` install the `IndifferentCoder` AS the attribute's type
(`store.rb:106-109`), so `typeForAttribute(storeAttribute).accessor()` resolves
for every declared store column. Then `storeAccessorFor` collapses to Rails'
single lookup + guard, and `_storeCoders` / `setStoreCoder` / `getStoreCoder`
delete along with it (three exported names of invented surface).

Note `getStoreCoder` currently walks the static prototype chain; the type route
inherits through `typeForAttribute` already, and PR #6427 confirmed the
subclass case resolves without the explicit declaring class.

## Acceptance criteria

1. `storeAccessorFor` is `type_for_attribute(store_attribute)` + the
   `respond_to?(:accessor)` guard + `.accessor`, and nothing else.
2. `_storeCoders`, `setStoreCoder` and `getStoreCoder` are gone (or the story is
   blocked with the specific reason the coder cannot become the type).
3. `store.test.ts` stays green, including the JSON/YAML coder arms and the
   subclass-inherited store column.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
