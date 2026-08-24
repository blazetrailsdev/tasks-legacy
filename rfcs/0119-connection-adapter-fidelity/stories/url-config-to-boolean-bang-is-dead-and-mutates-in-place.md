---
title: "UrlConfig#to_boolean! port has no callers and mutates in place"
status: closed
updated: 2026-08-24
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 25
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Premise is false on main (152b2ebe9): `toBooleanBang` has two live callers — database-configurations/url-config.ts:47-48 (`replica`, `databaseTasks`) — so it is neither dead nor never-run, and the frozen-hash TypeError the story predicted does not occur. If in-place mutation of a frozen config is still a concern it needs re-filing against the live call sites."
---

## Context

`toBooleanBang` (`packages/activerecord/src/database-configurations/url-config.ts:131-135`)
is exported with an `@internal` tag and mirrors Rails'
`UrlConfig#to_boolean!` (`vendor/rails/activerecord/lib/active_record/database_configurations/url_config.rb`),
but has **no callers anywhere in the repo**. The coercions it would perform are
done inline by `normalizeUrlHash` (url-config.ts:63-84) instead.

It mutates its argument in place. That is harmless today only because it never
runs — every config hash is frozen at construction as of PR #5509
(`database-config.ts`, mirroring `hash_config.rb:38-41`), so the first caller
to pass a config hash would get `TypeError: Cannot assign to read only
property`.

Surfaced in review of PR #5509.

## Acceptance criteria

- [ ] Either `normalizeUrlHash` routes its boolean coercions through
      `toBooleanBang` (so the ported helper is actually the implementation,
      matching Rails' `to_boolean!` call sites), or the dead export is removed.
- [ ] No in-place mutation of a hash that has been handed to a
      `DatabaseConfig` constructor.
- [ ] `pnpm parity:api` extra-surface totals do not regress.
