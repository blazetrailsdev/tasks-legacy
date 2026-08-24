---
title: "compute_if_absent returns per-row copies where Rails returns the stored entry"
status: draft
updated: 2026-08-23
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `Store#compute_if_absent` in PR #6929 (RFC 0096 wave-5
residual findings). The body now mirrors
`activerecord/lib/active_record/connection_adapters/abstract/query_cache.rb:66-80`
branch for branch, but one deviation was left standing and is not tracked
anywhere.

Rails hands back the **stored object itself** on both the hit path and the
store path:

```ruby
if entry = @map.delete(key)
  return @map[key] = entry          # query_cache.rb:70-72
end
...
@map[key] ||= yield                 # query_cache.rb:79
```

trails returns a shallow per-row copy at all three return sites
(`packages/activerecord/src/connection-adapters/abstract/query-cache.ts`):

```ts
return Promise.resolve(entry.map((row) => ({ ...row })));
...
if (stored) return stored.map((row) => ({ ...row }));
this._map.set(key, result);
return result.map((row) => ({ ...row }));
```

This predates PR #6929 — that PR ported the LRU touch, the `@map.shift`
eviction, and the `||=` store race, and deliberately left the copy alone rather
than widen its scope.

The copy is observable, not merely defensive: a caller that mutates a returned
row does not affect a later cache hit in trails, where in Rails it would. It
also allocates a fresh object per row on every cache hit, which is the hot path
the query cache exists to make cheap.

## Converged shape

`computeIfAbsent` returns the stored array itself on the hit path, the
race-loser path, and the store path — no `.map((row) => ({ ...row }))` — so
callers share the cached rows exactly as Rails' callers do.

The reason the copy exists needs establishing first: if a trails caller mutates
rows it reads out of the cache, that caller is the bug and converging here
requires fixing it too. Check `select_all` / `internal_exec_query` consumers
before removing the copy.

## Acceptance criteria

1. The three return sites in `computeIfAbsent` hand back the stored rows without
   copying, matching `query_cache.rb:70-72` and `:79`.
2. Any caller that relied on receiving a private copy is fixed rather than
   accommodated, or the copy is shown to be a genuine TypeScript language
   shortcoming and carries a `@missingRailsArgs` / `@noRailsEquivalent` receipt
   with a `PERMANENT` reason at the call site.
3. A test pins the shared-identity behaviour and fails on the current baseline.
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls` and
   `pnpm parity:api:calls:args` stay green.
