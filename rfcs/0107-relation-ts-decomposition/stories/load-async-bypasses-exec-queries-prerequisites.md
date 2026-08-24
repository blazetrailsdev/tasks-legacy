---
title: "loadAsync issues its query before execQueries' trails-only prerequisites"
status: blocked
updated: 2026-08-24
rfc: "0107-relation-ts-decomposition"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: 7
pr: 6906
claim: "2026-08-23T11:12:29Z"
assignee: "wave-5g-head-sweep"
blocked-by: "Re-verified against origin/main 2026-08-24: blocker still live. execMainQuery is still deliberately non-async (relation.ts:1125, with the FutureResult-adoption rationale in the doc comment at :1120-1124 — the body's :1115-1120 anchor has drifted by ~5 lines), so awaiting the prerequisites before execMainQuery(true) would still force every scheduled relation into the Promise arm and lose cancel universally. The hazard itself is unchanged and still latent: execQueries runs ensureSchemaLoaded + _materializeDeferredDistinctPkPredicates only on the foreground pass (relation.ts:1066; _materializeDeferredDistinctPkPredicates is still async at :1857). Unblocks only when both prerequisites leave the query path — schema warm before either entry point (RFC 0031 is closed, so this needs a new home) and distinct-PK materialization moved to where .where() puts it (finder_methods.rb:463-475)."
closed-reason: null
---

# `loadAsync` issues its query before `execQueries`' trails-only prerequisites

## Context

Since `split-future-result-scheduled-dispatch-out-of-exec-queries` (PR #6750),
`loadAsync` calls `execMainQuery` directly, exactly as Rails does
(`vendor/rails/activerecord/lib/active_record/relation.rb:1142`). Rails can do
that because `exec_main_query` (`relation.rb:1423-1452`) needs nothing from
`exec_queries` first.

trails' `execQueries` (`packages/activerecord/src/relation.ts`) does two
trails-only things BEFORE it reaches `execMainQuery`, neither of which has a
Rails counterpart:

- `ensureSchemaLoaded()` — lazy schema reflection, so callers need not call
  `loadSchema` explicitly.
- `_materializeDeferredDistinctPkPredicates()` — materializes Rails'
  `distinct_relation_for_primary_key` subquery into a literal id list, deferred
  to here because trails' `.where()` is synchronous.

Before PR #6750 the async path reached the query through
`toArray` -> `execQueries`, so both ran first. Now the `loadAsync` path issues
SQL without them, and they run only on the later foreground pass, where the
rows have already been fetched.

Two consequences, in increasing severity:

1. The SQL is built before lazy schema reflection, so a `loadAsync` that is the
   very first query against a model builds its Arel (and casts its bind values)
   against an unreflected schema.
2. `_materializeDeferredDistinctPkPredicates` exists so an empty id set becomes
   an empty `IN` — a contradiction that short-circuits before any SELECT
   (`relation.rb:1432-1433`). On the `loadAsync` path that materialization has
   not happened yet, so the short-circuit cannot fire.

Neither reproduced in the suites exercised at the time (`Topic`/`Book` cold-path
probes both passed, and the fixture harness always lays the canonical schema, so
the schema is warm in practice). Filed as a latent ordering hazard found by
inspection, not a live failure.

## Converged shape

The Rails-faithful direction is to make the two prerequisites unnecessary on
the query path rather than to re-thread them through `loadAsync`:

- Schema reflection should be satisfied before a relation can reach either
  entry point, not lazily inside one of them — this overlaps the schema-cache
  warming work, so check that before writing code here.
- The deferred distinct-PK materialization should happen where `.where()` puts
  it in Rails, not at load time.

If neither can move in this story, the fallback is to run both prerequisites on
the `loadAsync` path too, so the two entry points agree — but that is a
same-shape-in-two-places outcome and should be justified over converging.

## Acceptance criteria

- [ ] `loadAsync()` and `toArray()` build identical SQL for the same relation,
      including when it is the first query against the model (regression test
      that fails on the baseline).
- [ ] A relation carrying a deferred distinct-PK predicate short-circuits
      identically on both entry points (`relation.rb:1432-1433`).
- [ ] No prerequisite is duplicated into two entry points without a call-site
      justification saying why it could not move.
- [ ] `parity:api` `relation.rb` -> `relation.ts` stays 401/401;
      `parity:api:calls` / `:args` / `:extra` clean.
