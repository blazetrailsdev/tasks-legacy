---
title: "converge-remaining-activerecord-copy-on-write-stores-onto-class-attribute"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6887
claim: "2026-08-22T22:33:40Z"
assignee: "converge-remaining-activerecord-copy-on-write-stores-onto-class-attribute"
blocked-by: null
closed-reason: null
---

# Converge the remaining activerecord copy-on-write stores onto classAttribute()

## Context

Follow-up to `converge-activerecord-copy-on-write-stores-onto-class-attribute`,
which converged the association/reflection registries
(`_reflections`, `_associations`) onto `classAttribute()` from
`@blazetrails/activesupport` and removed `_ensureOwnAssociations` from
`associations/builder/association.ts` and
`associations/builder/has-and-belongs-to-many.ts`.

### Inventory: the 68 `class_attribute` declarations in `vendor/rails/activerecord/lib/`

Measured 2026-08-20 (`grep -rn "class_attribute :" vendor/rails/activerecord/lib/`).
Three buckets:

**A. Scalar-valued, already correct as a plain `static` field (no work).** A
plain JS `static x = v` on `Base` already gives Rails' `class_attribute` read
and write semantics: a subclass read walks the constructor chain, and
`Sub.x = v` creates an own property (local write). These need no change and are
NOT a divergence:

| Rails                                                                                                                                                                                                                                                                               | trails              |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| `integration.rb:16` `cache_timestamp_format`                                                                                                                                                                                                                                        | `base.ts:955`       |
| `integration.rb:24` `cache_versioning`                                                                                                                                                                                                                                              | `base.ts:954`       |
| `integration.rb:32` `collection_cache_versioning`                                                                                                                                                                                                                                   | `base.ts:956`       |
| `core.rb:24` `_destroy_association_async_job`                                                                                                                                                                                                                                       | `base.ts:4023`      |
| `core.rb:91` `strict_loading_by_default`                                                                                                                                                                                                                                            | `base.ts:3890`      |
| `core.rb:94` `has_many_inversing`                                                                                                                                                                                                                                                   | `base.ts:952`       |
| `inheritance.rb:47` `store_full_sti_class`                                                                                                                                                                                                                                          | `base.ts:3932`      |
| `model_schema.rb:164` `table_name_prefix`                                                                                                                                                                                                                                           | `base.ts:957`       |
| `model_schema.rb:168` `pluralize_table_names`                                                                                                                                                                                                                                       | `base.ts:4049`      |
| `model_schema.rb:169` `implicit_order_column`                                                                                                                                                                                                                                       | `base.ts:4044`      |
| `reflection.rb:13` `automatic_scope_inversing`                                                                                                                                                                                                                                      | `base.ts:950`       |
| `reflection.rb:14` `automatically_invert_plural_associations`                                                                                                                                                                                                                       | `base.ts:951`       |
| `timestamp.rb:47` `record_timestamps`                                                                                                                                                                                                                                               | `base.ts:1726`      |
| `attribute_methods/serialization.rb:20` `default_column_serializer`                                                                                                                                                                                                                 | `base.ts:2131`      |
| `attribute_methods/time_zone_conversion.rb:60` `time_zone_aware_attributes`                                                                                                                                                                                                         | `base.ts:967`       |
| `attribute_methods/dirty.rb:49-50` `partial_updates` / `partial_inserts`                                                                                                                                                                                                            | `base.ts:1740-1741` |
| plus `core.rb:22,47,87,89,92,96,98,100,102,104,119`, `inheritance.rb:43`, `model_schema.rb:163,166,167,170,172`, `signed_id.rb:13`, `locking/optimistic.rb:56`, `log_subscriber.rb:7`, `sqlite3_adapter.rb:67`, `abstract_mysql_adapter.rb:29`, `postgresql_adapter.rb:105,123,132` | same shape          |

**B. Container-valued (`[]` / `{}` / `Set.new`) — the ones where the plain-static
spelling is only safe as long as nothing mutates in place.** A grep for
`.push` / `.add(` / `[k] =` / `.delete` against each trails spelling found no
in-place mutation as of 2026-08-20, so these are currently correct — but they
carry no guard, and the next writer who reaches for `.push` silently mutates
the parent's container. Converting these to `classAttribute()` makes the
contract enforced rather than incidental:

- `reflection.rb:12` `aggregate_reflections` → `reflection.ts:2219`, still a
  `hasOwnProperty` copy-on-write over `_aggregateReflections`
- `readonly_attributes.rb:11` `_attr_readonly` → `readonly-attributes.ts:44`,
  `hasOwnProperty` copy-on-write over `_readonlyAttributes`
- `counter_cache.rb:9-10` `_counter_cache_columns` /
  `counter_cached_association_names` → `counter-cache.ts:315`
- `scoping/default.rb:19-20` `default_scopes` / `default_scope_override` →
  `base.ts:2163`
- `nested_attributes.rb:15` `nested_attributes_options`,
  `normalization.rb:8` `normalized_attributes`, `token_for.rb:10-11`
  `token_definitions` / `generated_token_verifier`,
  `encryption/encryptable_record.rb:11` `encrypted_attributes`,
  `attribute_methods/time_zone_conversion.rb:61-62` — no trails `static`
  counterpart found; audit each for where the state actually lives.
- `test_fixtures.rb:31-38` (8 declarations) → `test-fixtures.ts`

**C. Not a `class_attribute` at all — leave alone.** `core.rb:226-235`
`connection_class` is a plain `@connection_class` ivar on the class singleton
(per-class, does NOT inherit); `base.ts`'s `connectionClass` getter mirrors it
correctly with `hasOwnProperty` and its comment was corrected to say so.

## Acceptance criteria

- [ ] Every bucket-B entry either becomes a `classAttribute()` declaration, or
      carries a call-site citation of the Rails construct it actually mirrors.
- [ ] No `hasOwnProperty` copy-on-first-write helper survives for a store whose
      Rails counterpart is a `class_attribute`.
- [ ] No new copy-on-first-write helper is introduced under any name.
- [ ] `pnpm parity:api` / `pnpm parity:test` deltas non-negative;
      `pnpm parity:api:calls` and `:args` clean, no reseed.
- [ ] Split across PRs if it exceeds the LOC ceiling — bucket B is ~8 clusters.
