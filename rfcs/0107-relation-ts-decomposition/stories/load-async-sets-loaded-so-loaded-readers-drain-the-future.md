---
title: "load-async-sets-loaded-so-loaded-readers-drain-the-future"
status: done
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps:
  - converge-relation-loaded-arm-readers-onto-seams
deps-rfc: []
est-loc: null
priority: 2
pr: 6918
claim: "2026-08-23T14:57:28Z"
assignee: "load-async-sets-loaded-so-loaded-readers-drain-the-future"
blocked-by: null
closed-reason: null
---

# `loadAsync` must set `@loaded` so the `loaded?` readers drain the future

## Context

Follow-up to `split-future-result-scheduled-dispatch-out-of-exec-queries`
(PR #6750), which gave trails Rails' `@future_result` / `scheduled?` dispatch:
`loadAsync` parks the rows handle in `_futureResult`, `isScheduled` reads it
(`relation.rb:1170-1172`), and `execQueries` drains it through the
`if scheduled?` arm (`relation.rb:1405-1409`).

One half of Rails' shape was deliberately left out of that PR and is still
divergent: Rails' `load_async` sets `@loaded = true` in the same breath as it
parks the future (`vendor/rails/activerecord/lib/active_record/relation.rb:1149`).
That is what makes the `loaded?` readers cooperate with a scheduled query:

- `size` (`relation.rb:353-359`) takes the `records.length` branch,
- `empty?` (`relation.rb:362-370`) takes `records.empty?`,
- `one?` (`relation.rb:399-405`) takes `records.one?`,
- `many?` likewise,

and each of those goes through `records` -> `load`, whose guard is
`if !loaded? || scheduled?` (`relation.rb:1179-1186`) — so the reader drains
`@future_result` instead of issuing a second query.

trails' `loadAsync` (`packages/activerecord/src/relation.ts`) does NOT set
`_loaded`, and its readers short-circuit on `_loaded` straight into `_records`
rather than routing through `records()`:

- `size()` — `relation.ts:866-869` (`if (this._loaded) return this._records.length`)
- `isEmpty()` — `relation.ts:876-879`
- `isMany()` — `relation.ts:917`
- `isOne()` — nearby, same shape

So `rel.loadAsync().size()` issues a brand-new `COUNT(*)` while the scheduled
query is still in flight, where Rails joins the future. Raised in review on
PR #6750.

## Why it was not fixed in PR #6750

Setting `_loaded = true` in `loadAsync` alone is a correctness regression, not a
fix: four readers of `_loaded` are **synchronous** and reach `_records`
directly, and a sync TS method cannot await the drain that Rails' `records`
does synchronously at `relation.rb:1180`:

- `inspect()` — `relation.ts:618`
- the `isLoaded` / `loaded` getters — `relation.ts:695`, `relation.ts:2644`
- `computeCacheVersion()` — `relation.ts:2941`

With `_loaded` true and `_records` still empty, each of those answers as though
the relation loaded to zero rows. Converging therefore means deciding, per
reader, how a scheduled-but-undrained relation answers a synchronous question —
which is the actual work here, and is why it is its own story.

## Converged shape

- `loadAsync` sets `_loaded` alongside `_futureResult` (`relation.rb:1149`).
- `toArray`/`load`'s guard becomes Rails' `!loaded? || scheduled?`
  (`relation.rb:1180`).
- The async `loaded?` readers (`size`, `isEmpty`, `isMany`, `isOne`) route
  through `records()` rather than reading `_records`, so the drain happens
  exactly where `relation.rb:355` puts it.
- The four sync readers get an explicit, justified-at-the-call-site answer for
  the scheduled case (drain-is-impossible is a genuine language shortcoming
  there; the guard shape is the decision to make).

## Acceptance criteria

- [ ] `loadAsync()` sets `_loaded`, as `relation.rb:1149` does.
- [ ] `toArray`/`load` guard on `!loaded? || scheduled?` (`relation.rb:1180`).
- [ ] `rel.loadAsync()` followed by `size()` / `isEmpty()` / `isOne()` /
      `isMany()` drains the parked future and issues NO second query — covered
      by a regression test that fails on the baseline.
- [ ] The four sync `_loaded` readers do not regress to "loaded with zero rows"
      for a scheduled relation; each carries a call-site justification.
- [ ] `parity:api` `relation.rb` -> `relation.ts` stays 401/401;
      `parity:api:calls` / `:args` / `:extra` clean.
