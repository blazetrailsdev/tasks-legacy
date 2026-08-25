---
title: "Converge abstract_mysql_adapter concurrency tests from adapter-level SQL to the model layer"
status: done
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: 7061
claim: "2026-08-25T18:50:35Z"
assignee: "pg-column-serial-identity-fields-are-public-where-rails-has-ivars"
blocked-by: null
closed-reason: null
---

## Context

PR #5121 ported `adapters/abstract_mysql_adapter/count_deleted_rows_with_lock_test.rb` and `nested_deadlock_test.rb` at the **adapter level** (raw SQL on two `Mysql2Adapter` connections, `packages/activerecord/src/adapters/abstract-mysql-adapter/count-deleted-rows-with-lock.test.ts` and `nested-deadlock.test.ts`), matching the precedent set by the PG sibling `adapters/postgresql/transaction-nested.test.ts`. Rails drives these through the model layer: `Bulb.unscoped.delete_all` / `Author.create!` (count_deleted_rows_with_lock_test.rb:11) and `Sample.transaction(requires_new:)` + `s1.lock!` / `s2.update` (nested_deadlock_test.rb:37-160). Deviations: (a) no model layer — `Model.transaction` nesting, `lock!`, `update`, `delete_all` are all hand-inlined SQL; (b) the count test uses `test_bulbs`/`test_authors` instead of the canonical `bulbs`/`authors` tables + `Bulb`/`Author` models (chosen to keep DROP/CREATE off shared canonical tables owned by parallel workers).

## Acceptance criteria

- Both files (and the PG sibling, same shape) rewritten through the model layer once `Model.transaction({requiresNew})` + `lock!` support two concurrent connection leases in tests, using canonical `Bulb`/`Author` models for the count test and an inline `samples` model (faithful — Rails creates it inline).
- parity:test stays ✓ for all files; tests still provoke the real deadlock/race.
