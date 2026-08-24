---
title: "Drop subclass state from Column#encodeWith to Rails' seven base keys"
status: closed
updated: 2026-08-24
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: "Misspecified: premise contradicted by vendor/rails. Rails DOES persist subclass state — postgresql/column.rb:50-61 encodes serial/identity/generated, sqlite3/column.rb:35-42 encodes auto_increment; only mysql/column.rb has no coder overrides, because it derives unsigned?/auto_increment?/virtual? from sql_type and sql_type_metadata.extra (:7-24). PG's oid/fmod are not dropped either: they delegate to sql_type_metadata (:7), which the base encode_with already persists, and array derives from sql_type (:37-39). So 'Rails' seven base keys' is not the target shape, and the fixtures-warm comparison is not the residual deviation. Re-specified as converge-column-subclass-coders-to-rails-per-subclass-key-sets (same RFC), which carries the measured breakage: dropping the keys today leaves the fields undefined, and sqlite3/column.ts:55, postgresql/column.ts:90 and :100 port Ruby nil?/present? as x !== null, so undefined flips them true and primary-keys.test.ts reds with NOT NULL constraint failed: subscribers.nick."
---

## Context

`Column#encodeWith` writes Rails' seven keys — `name`, `sql_type_metadata`,
`null`, `default`, `default_function`, `collation`, `comment` — and nothing
else (`activerecord/lib/active_record/connection_adapters/column.rb:55-63`).
Rails persists **no subclass state**: YAML restores the adapter's `Column`
subclass from its `!ruby/object:` tag and `init_with`
(`column.rb:46-53`) fills only those seven base ivars, leaving
`PostgreSQL::Column#array`, `#serial`, `#oid`, `#identity`, `#generated`,
`MySQL::Column#auto_increment`, `SQLite3::Column#rowid` and co. nil.

trails' subclass `encodeWith` / `initWith` overrides go one step further and
encode their own state
(`packages/activerecord/src/connection-adapters/postgresql/column.ts`,
`mysql/column.ts`, `sqlite3/column.ts`, base at
`connection-adapters/column.ts:153-180`). #6980 converged the naming and the
class-discriminator question but deliberately left this arm: reproducing
Rails' key set exactly reintroduces the `base_test.rb` `test_clear_cache!`
failure, because trails' fixtures warm compares a **dump-loaded** schema cache
against a **reflected** one, where Rails only ever compares reflected against
reflected.

So the residual deviation is not the encoding — it is that trails' cache
comparison is stricter than Rails'. This story converges that.

## Acceptance criteria

- The subclass `encodeWith` / `initWith` overrides stop carrying subclass
  state, matching `column.rb:55-63`'s seven keys exactly — or the deviation is
  shown to be forced by a genuine TypeScript/JSON shortcoming and blocked with
  the specific blocker.
- The fixtures-warm path no longer compares a dump-loaded cache against a
  reflected one in a way Rails does not, so `test_clear_cache!` stays green
  without the extra keys.
- The deviation comment at `connection-adapters/column.ts:153-172` goes away
  with it.
