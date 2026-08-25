---
title: "Port TestFixtures' saved_pool_configs snapshot and restore"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::TestFixtures#setup_fixtures`
(`vendor/rails/activerecord/lib/active_record/test_fixtures.rb:112-160`) seeds
`@saved_pool_configs = Hash.new { |hash, key| hash[key] = {} }`
(`test_fixtures.rb:123`), the per-role/per-shard snapshot `setup_shared_connection_pool`
fills and `teardown_fixtures` restores so a test that repoints a connection
pool cannot leak that repointing into the next test.

Surfaced by RFC 0106 story `converge-remaining-default-block-hash-sites`
(PR #6870), which converged the other default-block Hash sites onto the settled
inline Proxy idiom. This one had no site to converge: `setup_fixtures` itself is
not ported. `packages/activerecord/src/test-fixtures/with-transactional-fixtures.ts`
covers the transaction open/rollback half of `setup_fixtures`/`teardown_fixtures`,
and nothing in `packages/activerecord/src` mentions `saved_pool_configs`, so the
pool-config save/restore half is simply absent.

## Converged shape

Port the `@saved_pool_configs` snapshot and its restore alongside the existing
transactional-fixtures wiring: `setup_shared_connection_pool`
(`test_fixtures.rb:236-252`) writes into it, `teardown_fixtures`
(`test_fixtures.rb:170-186`) walks it back through
`ActiveRecord::Base.connection_handler.set_pool_config(role, shard, pool_config)`.
Spell the hash itself with the settled inline Proxy `get` trap that vivifies a
missing string key (see `buildJoinBuckets` in
`packages/activerecord/src/relation/query-methods.ts`, and `PoolManager`'s
`_roleToShardMapping` from PR #6870) — no shared helper, since Ruby has none.

Note `PoolManager.setPoolConfig` / `getPoolConfig` are already Rails-faithful as
of PR #6870, so the restore path has what it needs.

## Acceptance criteria

- [ ] `savedPoolConfigs` exists on the trails fixtures path, spelled with the
      inline Proxy idiom and citing `test_fixtures.rb:123` at the site.
- [ ] It is populated where Rails populates it and restored where Rails restores
      it, mirroring `test_fixtures.rb:236-252` and `:170-186`.
- [ ] No shared `defaultHash` helper; no new `parity:api:extra` surface.
- [ ] Existing fixtures suites stay green on SQLite, PostgreSQL and MySQL/MariaDB.
