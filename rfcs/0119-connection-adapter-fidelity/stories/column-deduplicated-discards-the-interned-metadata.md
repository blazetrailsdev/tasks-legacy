---
title: "Column#deduplicated discards the interned sqlTypeMetadata, and SqlTypeMetadata#deduplicated never freezes"
status: ready
updated: 2026-08-25
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

Surfaced by review on PR #7047 (RFC 0119,
`column-bypasses-deduplicable-registry`) AFTER the branch had been pushed. The
fix was written and verified locally but **never landed** — #7047 merged with
the bug still in it. `main` at 6b045e0db carries it.

Two omissions, both of the same kind: a `Deduplicable` step that the port drops.

### 1. `Column#deduplicated` discards the deduplicated metadata

`vendor/rails/activerecord/lib/active_record/connection_adapters/column.rb:107`:

```ruby
@sql_type_metadata = sql_type_metadata.deduplicate if sql_type_metadata
```

Rails **assigns the return value back**. `Deduplicable#deduplicate`
(`deduplicable.rb:18`) is `registry[self] ||= deduplicated`, so it yields the
ALREADY-REGISTERED canonical instance when one exists — a _different_ object
than the receiver.

`packages/activerecord/src/connection-adapters/column.ts` calls it for effect
and throws the result away:

```ts
deduplicated(): this {
  if (this.sqlTypeMetadata) {
    this.sqlTypeMetadata.deduplicate();   // <- return value discarded
  }
  return Object.freeze(this);
}
```

So whenever an equal-but-distinct `SqlTypeMetadata` was already registered, the
column keeps pointing at its own non-canonical copy. Two columns that correctly
dedup to one `Column` (the key is content-based) still each carry their own
metadata object, defeating the interning for that field and leaving an
unfrozen object reachable from a frozen `Column`.

### 2. `SqlTypeMetadata#deduplicated` drops the mixin's `freeze`

`Deduplicable#deduplicated` is `freeze` (`deduplicable.rb:26`), and
`SqlTypeMetadata` adds nothing to it — unlike `Column#deduplicated`, which
interns its strings and then calls `super`. trails'
`connection-adapters/sql-type-metadata.ts` returns the receiver unfrozen:

```ts
deduplicated(): this {
  return this;
}
```

## Converged shape

```ts
// column.ts — assign back, before the freeze
if (this.sqlTypeMetadata) {
  this.sqlTypeMetadata = this.sqlTypeMetadata.deduplicate();
}
return Object.freeze(this);

// sql-type-metadata.ts
deduplicated(): this {
  return Object.freeze(this);
}
```

Both were verified locally against the full adapter, schema, dumper, model and
association suites (4552 tests) with no fallout — every `SqlTypeMetadata` field
is `readonly`, so freezing is safe. The saved patch is 61 lines including the
test.

## Acceptance criteria

- [ ] `Column#deduplicated` assigns `sqlTypeMetadata.deduplicate()` back, ahead
      of the freeze (`column.rb:107`).
- [ ] `SqlTypeMetadata#deduplicated` freezes (`deduplicable.rb:26`).
- [ ] A test asserts two columns built from separately-constructed but equal
      metadata share ONE frozen `sqlTypeMetadata` instance, failing on baseline.
      (`column-equality.trails.test.ts` >
      `interns the sqlTypeMetadata onto the shared canonical instance` is the
      written form; it fails on `main` today.)
- [ ] Adapter, schema-cache, dumper and association suites stay green on all
      three lanes.
