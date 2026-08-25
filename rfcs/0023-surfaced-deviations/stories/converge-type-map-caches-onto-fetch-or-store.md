---
title: "Converge the two type-map caches onto Rails' fetch_or_store shape"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# Converge the two type-map caches onto Rails' `fetch_or_store` shape

## Context

Surfaced while converging `perform_fetch` in #6824 (RFC 0106). Both type maps
cache correctly-shaped results but reach them through machinery Rails does not
have, and one of them is a live truthiness trap.

### 1. `TypeMap#fetch` re-fetches a falsy cached value

`vendor/rails/activerecord/lib/active_record/type/type_map.rb:18-22`:

```ruby
def fetch(lookup_key, &block)
  @cache.fetch_or_store(lookup_key) do
    perform_fetch(lookup_key, &block)
  end
end
```

`fetch_or_store` recomputes only when the key is ABSENT. trails
(`packages/activerecord/src/type/type-map.ts:23-28`) writes:

```ts
const cached = this._cache.get(lookupKey);
if (cached) return cached;
```

which recomputes whenever the stored value is falsy. This is the CLAUDE.md
"Ruby truthiness" trap and the `fetch` vs `??` trap in one line. It is inert
today only because every stored value is a `Type` instance; it is wrong the
moment one is not, and it reads as a cache-hit test when it is a truthiness
test. Converged shape: `has()`-then-`get()` (or a sentinel), so absence — not
falsiness — is what triggers `performFetch`.

### 2. `HashLookupTypeMap#fetch` hand-rolls an args cache key

`vendor/rails/activerecord/lib/active_record/type/hash_lookup_type_map.rb:13-21`
keys the inner cache on the `args` Array itself, because `Concurrent::Map`
hashes an Array structurally:

```ruby
@cache = Concurrent::Map.new do |h, key|
  h.fetch_or_store(key, Concurrent::Map.new)
end

def fetch(lookup_key, *args, &block)
  @cache[lookup_key].fetch_or_store(args) do
    perform_fetch(lookup_key, *args, &block)
  end
end
```

trails (`packages/activerecord/src/type/hash-lookup-type-map.ts:22-84`) instead
builds a ~40-line string key by hand: a `parts` array with `\x00undef`,
`\x00null`, `\x00bigint:`, `\x00symbol:`, `\x00fn:` sentinels, a
`JSON.stringify` fallback, a `cacheable` flag that bails to an uncached
`performFetch` on a throw, and a `\x01` joiner. None of that has a Rails
counterpart — it is a structural-hash-key emulation, and it also carries the
same falsy-cache-hit bug as (1) (`if (cached) return cached`).

The same file also splits the trailing-block detection by hand
(`typeof rest[rest.length - 1] === "function"`) where Rails takes `&block`.

## Acceptance criteria

- [ ] Neither `fetch` treats a falsy stored value as a cache miss; absence is
      the only thing that triggers `performFetch` (Rails `fetch_or_store`).
- [ ] `HashLookupTypeMap`'s bespoke `parts` / `cacheable` / sentinel-prefix key
      construction is retired in favour of one structural-key helper (or a
      nested `Map` keyed per positional arg), with no per-type sentinel list.
- [ ] `pnpm parity:api:extra --package activerecord` shows no new novel surface
      for either file, and ideally fewer.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green; no
      new baseline rows.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green — the PG OID and MySQL
      extended type maps are the heaviest users of the args-keyed cache.
