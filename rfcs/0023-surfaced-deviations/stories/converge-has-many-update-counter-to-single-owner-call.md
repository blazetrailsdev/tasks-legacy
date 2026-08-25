---
title: "Drop the three-way owner probe in has_many update_counter for Rails' single increment! call"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced by PR #6367, which converged the counter-cache _column_ lookup in
`HasManyAssociation` onto the reflection but left the write dispatch alone.

Rails (`activerecord/lib/active_record/associations/has_many_association.rb:98-110`):

```ruby
def update_counter(difference, reflection = reflection())
  if reflection.has_cached_counter?
    owner.increment!(reflection.counter_cache_column, difference)
  end
end

def update_counter_in_memory(difference, reflection = reflection())
  if reflection.counter_must_be_updated_by_has_many?
    counter = reflection.counter_cache_column
    owner.increment(counter, difference)
    owner.send(:"clear_#{counter}_change")
  end
end
```

One call each. trails' `updateCounter`
(`packages/activerecord/src/associations/has-many-association.ts:363-380`)
instead probes three owner methods in turn:

```ts
if (typeof owner.incrementBang === "function") {
  await owner.incrementBang(column, difference);
} else if (typeof owner.updateCounters === "function") {
  await owner.updateCounters({ [column]: difference });
} else if (typeof owner.increment === "function") {
  owner.increment(column, difference);
}
```

Two of the three arms are unreachable for a real `Base` subclass — which always
has `incrementBang` — so they exist only for plain-object test doubles, and
they silently substitute a _different_ Rails method (`update_counters`,
non-bang `increment`) when the first probe misses. `updateCounterInMemory` has
the matching shape: `owner.readAttribute` / `writeAttribute` /
`clearAttributeChange` are each optional-chained rather than called, where
Rails sends `increment` and `clear_#{counter}_change` unconditionally.

## Converged shape

Both bodies make the single Rails call, unguarded:
`owner.incrementBang(reflection.counterCacheColumn(), difference)` and
`owner.increment(counter, difference)` + the counter's `clear…Change`. Any test
double that relied on a fallback arm gets a real model or the missing method.

## Acceptance criteria

- [ ] `updateCounter` / `updateCounterInMemory` each make one unguarded call,
      matching `has_many_association.rb:98-110`.
- [ ] No `typeof owner.X === "function"` probes or optional-call chains remain
      in either body.
- [ ] Any test double that depended on a removed arm is converted to a
      canonical model rather than reinstating the probe.
- [ ] `pnpm parity:api:calls` / `:args` green; retired rows deleted by hand
      (only-shrink, no `--write`). AR suites pass on all three adapter lanes.
