---
title: "Relation#updateCounters should take a single counters arg, not a second options param"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging the empty-updates behavior in #4992.

Rails' `Relation#update_counters` takes a SINGLE argument and extracts `:touch`
from the counters hash itself (relation.rb:926-927):

```ruby
def update_counters(counters)
  touch = counters.delete(:touch)
```

trails adds a second parameter that Rails does not have
(`packages/activerecord/src/relation.ts:6074-6077`):

```ts
async updateCounters(
  counters: Record<string, number | { time?: Temporal.Instant }>,
  options?: { touch?: CounterCacheTouchOption },
): Promise<number>
```

The implementation already reads the Rails-native in-hash form first —
`const touchOption = (touchFromCounters ?? options?.touch)` — so the extra
parameter is a pure additive API-surface deviation, not a behavior difference.

The class-level `Model.updateCounters(id, counters)` (counter_cache.rb:115) is
also two-arg in Rails, so only the Relation surface diverges.

## Call sites to reroute

Removing the parameter means moving `touch` into the counters hash at:

- `associations/belongs-to-association.ts:172` — `scope.updateCounters({ [counterCol]: by }, opts)`
- `associations/belongs-to-association.ts:485` — same shape
- `counter-cache.ts:71` — `relation.updateCounters(counters, options)`

Note the counters hash is typed `Record<string, number | { time?: Instant }>`;
admitting a `touch` key means widening that value union to include
`CounterCacheTouchOption`, mirroring Ruby's untyped hash.

## Acceptance criteria

- [ ] `Relation#updateCounters` takes a single `counters` argument, matching
      relation.rb:926.
- [ ] `touch` is read only via the in-hash form (`counters.touch`), as Rails
      does with `counters.delete(:touch)` — including that the key is REMOVED
      before the counter loop, so it never becomes an update column.
- [ ] The three internal call sites above pass `touch` inside the hash.
- [ ] Existing counter-cache touch coverage stays green (counter-cache.test.ts,
      belongs-to-associations.test.ts).
