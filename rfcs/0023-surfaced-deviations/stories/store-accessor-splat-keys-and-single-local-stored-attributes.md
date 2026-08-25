---
title: "store_accessor takes an options object where Rails splats keys; local_stored_attributes has two TS names and no writer"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 180
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by PR #6844, which rebuilt `storeAccessor`
(`packages/activerecord/src/store.ts`) into Rails' single-method shape. Two
signature-level divergences were left in place because converging them touches
every caller and was out of that PR's scope.

Rails (`activerecord/lib/active_record/store.rb:112-113, 199-205`):

```ruby
def store_accessor(store_attribute, *keys, prefix: nil, suffix: nil)
  keys = keys.flatten
  ...
end
```

```ruby
included do
  class << self
    attr_accessor :local_stored_attributes
  end
end
```

**1. `*keys` splat + `keys.flatten`.** trails takes an options object:

```ts
export function storeAccessor(
  modelClass: typeof Base,
  storeAttribute: string,
  options: { accessors?: string[]; prefix?: boolean | string; suffix?: boolean | string },
): void;
```

so `store_accessor :settings, :privileges, :servants` — the documented spelling
in store.rb's own class comment — has no trails equivalent, and the `flatten`
that lets `store` forward `options[:accessors]` (an Array) into the same splat
(store.rb:109) has nothing to flatten. Rails' `prefix:`/`suffix:` really are
kwargs; only `keys` should move.

**2. Two TS functions for one Ruby `attr_accessor`.** `localStoredAttributes`
takes the class as a parameter; `localStoredAttributesMethod` is a `this`-typed
wrapper wired onto `Base` (`base.ts`, `declare static localStoredAttributes`).
Rails has exactly one name. `parity:api:extra --package activerecord` scores
`localStoredAttributesMethod` as novel surface in `store.ts`. There is also no
writer half — Rails' `attr_accessor` gives `self.local_stored_attributes =`,
which `store_accessor` uses (store.rb:182-184); trails reaches into the
`_storedAttributes` WeakMap directly at that site instead.

## Converged shape

```ts
export function storeAccessor(
  this: typeof Base,
  storeAttribute: string,
  ...keys: (string | string[])[]
): void;
```

with `prefix`/`suffix` as the trailing kwargs object the repo already uses for
Ruby kwargs, `keys.flat()` as `keys.flatten`, and `store` (store.rb:109)
forwarding `options.accessors` into it. Callers are `store` and `store.test.ts`.

For the reader: collapse to the single Rails name via the `this`-typed-function
mixin idiom (CLAUDE.md, "Module mixins"), so `Base.localStoredAttributes` and
the module-level export are one function, and add the writer half
(`setLocalStoredAttributes`, per CLAUDE.md's `setX()` rule for Ruby `x=`) so
`store_accessor`'s tail assignment goes through it rather than the WeakMap.

## Acceptance criteria

- [ ] `storeAccessor` takes `storeAttribute` then a rest-parameter of keys,
      flattened, matching store.rb:112-113.
- [ ] `store` forwards `options.accessors` into the splat (store.rb:109).
- [ ] One TS name for `local_stored_attributes`, plus a `setX` writer;
      `localStoredAttributesMethod` is gone from
      `pnpm parity:api:extra --package activerecord`.
- [ ] `store_accessor`'s tail assignment (store.rb:182-184) goes through the
      writer, not `_storedAttributes` directly.
- [ ] `store.test.ts` green on SQLite, PostgreSQL and MySQL/MariaDB.
