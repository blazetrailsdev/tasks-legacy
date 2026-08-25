---
title: "Column bypasses the Deduplicable registry and has no deduplicateKey"
status: done
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
deps: []
deps-rfc: []
est-loc: 140
priority: null
pr: 7047
claim: "2026-08-25T16:18:38Z"
assignee: "collection-proxy-association-seat-is-degenerate-for-singular-names"
blocked-by: null
closed-reason: null
---

## Context

`ActiveRecord::ConnectionAdapters::Column` does `include Deduplicable`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/column.rb:8`),
so `Column.new` is wrapped by `Deduplicable::ClassMethods#new`
(`deduplicable.rb:13`) and every construction goes through
`registry[self] ||= deduplicated` (`deduplicable.rb:18`). Identical columns
therefore collapse to one frozen instance. The registry key is the column
itself, which is why Rails pairs `==`/`eql?` with `hash` (`column.rb:75`/`:87`) —
those three are what make the `Hash` lookup work.

trails' `Column` (`packages/activerecord/src/connection-adapters/column.ts`)
overrides the mixin instead:

```ts
deduplicate(): this {
  return this.deduplicated();
}
```

It never touches the Deduplicable registry, and it does not implement
`deduplicateKey()` — the key the trails port of `deduplicate`
(`connection-adapters/deduplicable.ts:31`) needs to build its Map key. So
`Column` silently opts out of dedup entirely: every reflected column is a
distinct object where Rails shares one.

This is the same bypass as
[[adapter-type-metadata-bypasses-deduplicable-registry]] (MySQL/PostgreSQL
`TypeMetadata`), but in a third class that story does not scope. `SqlTypeMetadata`
is the one member of the cluster that does implement `deduplicateKey`
(`sql-type-metadata.ts`) and goes through the shared path.

PR #5630 ported `Column#==` as `equals`, which supplies the value-equality half
of what the registry key needs; `deduplicateKey` is the remaining piece.

Note that Rails' `Column#deduplicated` also interns the string attributes
(`column.rb:104-112`: `-name`, `-default`, `-default_function`, `-collation`,
`-comment`) and calls `super` to `freeze`. trails' `deduplicated` only forwards
`deduplicate` to `sqlTypeMetadata` and never freezes.

## Acceptance criteria

- [ ] `Column` implements `deduplicateKey()` over the attributes `==`/`hash`
      compare (`column.rb:75`/`:87`), and drops the `deduplicate()` override so
      construction goes through the shared registry path in
      `connection-adapters/deduplicable.ts`.
- [ ] The PostgreSQL and SQLite3 subclasses extend the key with the attributes
      their own `hash` folds in (`postgresql/column.rb:72`, `sqlite3/column.rb:53`
      — note the SQLite one includes `rowid`, which its `==` does not).
- [ ] `deduplicated` freezes, matching `Deduplicable#deduplicated`
      (`deduplicable.rb:26`), or the omission is justified at the call site if
      freezing breaks existing callers.
- [ ] A test asserts two identically-constructed columns are the same instance,
      failing on baseline.
- [ ] `parity:api` / `parity:api:extra` deltas stay non-negative.

## Absorbed: `adapter-type-metadata-bypasses-deduplicable-registry`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "MySQL/PostgreSQL TypeMetadata bypass the Deduplicable registry"

### Context

`MySQL::TypeMetadata#deduplicate` and `PostgreSQL::TypeMetadata#deduplicate`
(`packages/activerecord/src/connection-adapters/mysql/type-metadata.ts:62`,
`connection-adapters/postgresql/type-metadata.ts:61`) each override the mixin
with `return this.deduplicated();` — they never touch the Deduplicable
registry. Rails has no such override: `Deduplicable#deduplicate` is
`self.class.registry[self] ||= deduplicated`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/deduplicable.rb:18`),
so every including class shares one registry and identical metadata objects
collapse to a single frozen instance.

The overrides exist because neither class implements `deduplicateKey()`, which
the trails port of `deduplicate`
(`connection-adapters/deduplicable.ts:31`) needs to build its Map key —
`SqlTypeMetadata` does implement it (`sql-type-metadata.ts:37`) and goes through
the shared path. So the two adapter subclasses silently opt out of dedup.

Surfaced by PR #5406, which routed `deduplicate` through `registry()` to match
Rails and had to baseline these two in the wide call gate
(`scripts/api-compare/call-mismatches-wide-exclude/activerecord/connection-adapters/{mysql,postgresql}/type-metadata.json`).

### Acceptance criteria

- Both `TypeMetadata` classes implement `deduplicateKey()` (they already have a
  `hashKey()` that enumerates exactly the right fields) and drop the
  `deduplicate` override so the mixin's registry path applies, as in Rails.
- The two wide-gate baseline entries for `deduplicate` -> `registry` are DELETED,
  not re-reasoned.
- `pnpm parity:api:calls` OK; adapter column/type-reflection tests green on
  mysql2 and postgresql.

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
