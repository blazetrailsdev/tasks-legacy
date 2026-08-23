---
title: "compute_if_absent drops Rails' LRU touch and @map.shift eviction"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

Surfaced while reading `compute_if_absent` against the Ruby for RFC 0096 wave 5
(PR #6917). The trails port is missing two behaviours of Rails' query-cache
store, so it is not an LRU cache at all — it evicts the _oldest inserted_ entry
regardless of use.

Rails (`activerecord/lib/active_record/connection_adapters/abstract/query_cache.rb`,
`Store#compute_if_absent`):

```ruby
def compute_if_absent(key)
  check_version

  return yield unless @enabled

  if entry = @map.delete(key)
    return @map[key] = entry
  end

  if @max_size && @map.size >= @max_size
    @map.shift # evict the oldest entry
  end

  @map[key] ||= yield
end
```

Two things trails omits
(`packages/activerecord/src/connection-adapters/abstract/query-cache.ts:76-98`):

1. **The LRU touch.** Rails `@map.delete(key)` then re-inserts, which moves a
   _hit_ to the end of the Ruby Hash's insertion order. trails reads with
   `this.get(key)` and leaves the entry where it is, so a hot key is evicted as
   readily as a cold one.
2. **`@map.shift` eviction.** Rails evicts the head of the hash, which — because
   of (1) — is the least-recently-_used_ entry. trails does
   `this._map.keys().next().value` and deletes it, which is the least-recently-
   _inserted_ entry. With (1) fixed, a JS `Map` re-insert gives the same
   ordering, so the two halves must land together.

The dropped `@map.delete(key)` is also why `query-cache.ts` still carries a
convergeable `naming` row against `delete` in the RFC 0096 population: the ts
`delete` call the recorder matches is the eviction one, not the LRU touch.

## Converged shape

`compute_if_absent` deletes and re-inserts on a hit, and evicts via the first
key only after the touch is in place, so both operations read off the same
`Map` insertion order Rails gets from its Hash.

## Acceptance criteria

1. `compute_if_absent` mirrors the Ruby body branch for branch: `check_version`,
   the `@enabled` early return, the delete-and-reinsert hit path, the
   `@max_size` eviction, then the compute.
2. A regression test proves the LRU behaviour — with `max_size` N, reading the
   oldest key and then inserting a new one evicts the _second_-oldest, not the
   one just read. It must fail on the current baseline.
3. The `naming` row for `query-cache.ts#compute_if_absent` leaves the
   convergeable population; no `call-mismatches-exclude/` row is added.
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls` and
   `pnpm parity:api:calls:args` stay green.
