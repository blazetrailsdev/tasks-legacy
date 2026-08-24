---
title: "Drive sharding separation tests through the model API, as Rails does"
status: draft
updated: 2026-07-28
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 100
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while doing PR #5493 (RFC 0029 file-based sharding databases).

`same shards across clusters` and `sharding separation` in
`packages/activerecord/src/connection-adapters/connection-handlers-sharding-db.test.ts`
drive the database with raw SQL strings through `leaseConnection()` —
`CREATE TABLE`, `INSERT INTO`, `SELECT ... WHERE shard_key = '...'` — and assert
on returned row arrays.

Rails drives both tests through the model API
(`vendor/rails/activerecord/test/cases/connection_adapters/connection_handlers_sharding_db_test.rb:361-408`):

- `test_same_shards_across_clusters` (rb:361-375) uses
  `ShardConnectionTestModel.create!(shard_key: "test_model_default")` and
  `ShardConnectionTestModel.where(shard_key: ...).first.shard_key`.
- `test_sharding_separation` (rb:377-408) uses `create!` and
  `find_by_shard_key("foo")` / `assert_not find_by_shard_key(...)`.

This matters because the point of both tests is that _the model_ resolves to
the right shard's connection. Going through `leaseConnection()` by hand and
issuing literal SQL bypasses the model's connection resolution, so the tests
assert far less than their Rails counterparts do — a regression in
`Model` → pool routing would not fail them.

Note both tests legitimately keep `":memory:"` (Rails does too, rb:362-363 and
rb:379-380) — this story is about the query surface, not the database paths.
Rails' table is `shard_connection_test_models (shard_key VARCHAR(255))`, created
inline per shard; it is not in the canonical schema, and the models are Rails'
own `ShardConnectionTestModel` / `ShardConnectionTestModelB` defined in the test
file (rb:339-351).

## Acceptance criteria

- [ ] Both tests create and read rows via the model API (`create!`, `where`,
      `findBy...`) rather than raw SQL through `leaseConnection()`.
- [ ] The assertions match Rails': `shard_key` value equality in
      `same shards across clusters`; present/absent record checks in
      `sharding separation`.
- [ ] Test names unchanged; `":memory:"` usage unchanged.
- [ ] Verify the new form actually fails if model-to-shard pool resolution is
      broken (regression test must fail on a deliberately broken baseline).
