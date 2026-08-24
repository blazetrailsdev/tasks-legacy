---
title: "Build sharding-db test pools via connectsTo, not handler.establishConnection"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while doing #5493 (RFC 0029 file-based sharding databases).

Four tests in
`packages/activerecord/src/connection-adapters/connection-handlers-sharding-db.test.ts`
build their pools by calling `Base.connectionHandler.establishConnection(new
HashConfig(...), { owner, role, shard })` directly. Rails builds all of them
from a `configurations` hash plus `connects_to(shards: {...})`:

- `switching connections via handler` — Rails
  `test_switching_connections_via_handler`
  (`vendor/rails/activerecord/test/cases/connection_adapters/connection_handlers_sharding_db_test.rb:106-164`)
  uses a `default_env` config hash + `connects_to`.
- `cannot swap shards while prohibited` — Rails rb:283-310, same shape.
- `can swap roles while shard swapping is prohibited` — Rails rb:312-336, same
  shape.
- `retrieve connection pool with invalid shard` — Rails rb:226-229 establishes
  no pool at all; it relies on the ambient `arunit` connection. trails
  establishes a `HashConfig` pool first.

The handler-direct path skips `connects_to`'s shard-key registration entirely,
which is why these tests then have to poke `(Base as any)._shardKeys = [...]`
by hand (see the `switching connections via handler` body). That hand-poking is
a tell that the test is not exercising the code path Rails exercises.

PR #5493 deliberately left this shape alone — it was scoped to database paths, and
converting the setup shape is a behavioural change to what the tests cover.

## Acceptance criteria

- [ ] The four tests above build pools via `Base.configurations(...)` +
      `Base.connectsTo({ shards: ... })`, mirroring their Rails counterparts.
- [ ] The manual `(Base as any)._shardKeys = [...]` assignments disappear
      (shard keys should come from `connectsTo`).
- [ ] `retrieve connection pool with invalid shard` establishes no pool of its
      own if the suite's ambient connection makes that possible; otherwise
      document why at the call site.
- [ ] Test names unchanged.
- [ ] Existing `dbConfig.database` / `dbConfig.name` assertions still pass.
