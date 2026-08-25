---
title: "counter-cache.ts re-parses touch through the trails-only parseCounterCacheTouch"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 110
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `counter-cache.ts` still parses `touch` through the trails-only `parseCounterCacheTouch`

## Context

PR 6563 converged `Relation#update_counters` to Rails' inline
`Array.wrap` + `extract_options!` (relation.rb:935-941) and dropped its use
of the trails-only helper `parseCounterCacheTouch`. One caller remains:
`packages/activerecord/src/counter-cache.ts:201`.

`parseCounterCacheTouch` lives at
`packages/activerecord/src/timestamp.ts:358-371` and has no Rails
counterpart — `pnpm parity:api:extra --package activerecord` scores it as
novel surface on `timestamp.ts` (alongside `parseTouchAllArgs` /
`parseTouchArgs`, which are the same class of helper).

Rails does this inline at each site. For the counter-cache path,
`ActiveRecord::CounterCache::ClassMethods#update_counters` is
`activerecord/lib/active_record/counter_cache.rb:162-170`, which hands the
whole hash — `touch` included — to
`unscoped.where!(primary_key => id).update_counters(counters)`, i.e. it
routes into the very `Relation#update_counters` body (relation.rb:926-945)
that already parses `touch` the Rails way after 6563. There is no second
parse in Rails at all.

## Converged shape

Route `counter-cache.ts`'s path into the already-converged
`Relation#updateCounters` rather than re-parsing `touch` beside it, per
counter_cache.rb:162-170. Once that caller is gone, delete
`parseCounterCacheTouch` from `timestamp.ts` — that is one novel name off
`api:extra`.

Check `reset_counters`' touch handling (counter_cache.rb:56-96) while in
here; it reads the same option and may share the helper.

## Acceptance criteria

- [ ] `parseCounterCacheTouch` has no callers and is deleted from
      `timestamp.ts`.
- [ ] The `{ time: }` hash form and `touch: []` (no names) behave as before —
      `counter-cache.test.ts` covers both.
- [ ] `pnpm parity:api:extra --package activerecord` novel count for
      `timestamp.ts` drops by one.
- [ ] All three adapter lanes green.
