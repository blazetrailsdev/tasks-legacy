---
title: "excluding must drain a scheduled relation instead of re-querying its ids"
status: done
updated: 2026-08-23
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: 6922
claim: "2026-08-23T15:42:31Z"
assignee: "excluding-must-drain-a-scheduled-relation"
blocked-by: null
closed-reason: null
---

# `excluding` must drain a scheduled relation instead of re-querying its ids

## Context

PR #6918 gave `load_async` Rails' `@loaded = true` alongside `@future_result`
(`vendor/rails/activerecord/lib/active_record/relation.rb:1149`).

Rails' `QueryMethods#excluding`
(`vendor/rails/activerecord/lib/active_record/relation/query_methods.rb`) folds
relation arguments with `relations.flat_map(&:ids)`, and `Calculations#ids`
takes its `loaded?` arm through the `records` seam
(`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:373`),
so a `load_async` relation contributes its DRAINED rows with no extra query.

trails' `excluding` (`packages/activerecord/src/relation/query-methods.ts:1970`)
is synchronous and reads `relation._records` directly, so #6918 had to spell the
arm as `relation.isLoaded && !relation.isScheduled`: a scheduled relation falls
to the deferred-marker path, which materializes its ids with a SECOND id-select
at load time. Correct results, one query Rails does not issue.

The `_records` read is the actual divergence — `ids` itself was converged onto
the seam in #6918 and already drains.

## Converged shape

Have `excluding` fold relation arguments through the `ids` marker path
uniformly, or defer the `loaded?` read to the point where the load pipeline
materializes the marker (which is already async and can `await
relation.records()`), so `loaded?` alone is the guard and a scheduled relation
contributes the drained rows. Drop the `!relation.isScheduled` conjunct and its
call-site comment.

## Acceptance criteria

- [ ] `excluding` guards relation arguments on `loaded?` alone.
- [ ] No read of `relation._records` survives in `query-methods.ts`.
- [ ] `Model.excluding(other.loadAsync())` issues no query beyond the scheduled
      one — regression test failing on the baseline.
- [ ] `parity:api:calls` / `:args` / `:extra` clean.
