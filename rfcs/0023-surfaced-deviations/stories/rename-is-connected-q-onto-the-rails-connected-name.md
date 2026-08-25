---
title: "Make isConnected the real port of connected?, not an alias of isConnectedQ"
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

Rails' `ActiveRecord::ConnectionHandling#connected?`
(`vendor/rails/activerecord/lib/active_record/connection_handling.rb`) maps to
`isConnected` under the repo's Ruby→TS predicate convention
(`scripts/api-compare/conventions.ts` `name.endsWith("?")` →
`[isPrefixed, camel]`). trails ports the body as `isConnectedQ`
(`packages/activerecord/src/connection-handling.ts:470`) and exposes
`export const isConnected = isConnectedQ` (`:478`) as an alias — the divergent
name is the real path, the Rails name is the alias, which is the inversion
[[project_rails_name_is_real_path_not_divergent_alias]] warns about.

Surfaced in PR #5377: converging `QueryCache::ClassMethods#cache`/`#uncached`
required calling `this.isConnected()` so the wide call gate could credit
`connected?`; every other caller still goes through `isConnectedQ`
(`base.ts:1595`, `base.ts:4626`, `test-setup-dy.ts:79`, plus the test suites).

## Acceptance criteria

- `isConnected` is the defined function; `isConnectedQ` is deleted (or kept
  only if some caller genuinely needs it, with the reason at the call site).
- Every in-tree caller, including tests, uses the Rails name.
- `packages/activerecord/src/connection-handling.test.ts`'s
  "#isConnected delegates to isConnectedQ" trails test goes away with the alias.
- `pnpm parity:api` coverage for `connection_handling.rb` does not regress and
  the wide call baseline does not grow.
