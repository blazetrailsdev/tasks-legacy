---
title: "primaryKeyValue has no Rails counterpart and returns a tuple no call site wants"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 180
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`CollectionAssociation#primaryKeyValue`
(`packages/activerecord/src/associations/collection-association.ts:1324`) has no
Rails counterpart. Rails' `CollectionAssociation` never extracts a
"primary key value" helper — every call site reads `record.id` directly and
shapes it the way that site needs:

- `include?` (`activerecord/lib/active_record/associations/collection_association.rb:258-270`,
  the key line at `:266`):
  `record_id = klass.composite_primary_key? ? klass.primary_key.zip(record.id).to_h : record.id`
  — a **column=>value hash** for a composite key, because that is what
  `scope.exists?` accepts.
- `find_by_scan` (`collection_association.rb:522-541`): `r.id.to_s` /
  `ids.include?(r.id.to_s)` — **stringified**, never a tuple.

trails' helper instead returns a scalar for a single PK and a **bare array** for
a composite one, which matches neither shape. It is also overridden in
`HasManyThroughAssociation#primaryKeyValue`
(`has-many-through-association.ts:249-256`) to read
`associationPrimaryKey()` instead — a second invented rung on the same
non-Rails abstraction.

This bit us in PR #6743: delegating `CollectionProxy#include?` to
`CollectionAssociation#isInclude` routed the composite case through
`primaryKeyValue`, whose bare tuple `buildWhereClause` rejects with an
`ArgumentError`. The proxy's duplicated body had been building the hash itself,
masking the gap. #6743 fixed the `include?` call site inline
(`collection-association.ts:816-824`, now doing Rails' zip-to-hash) but left the
helper and its remaining callers in place.

Remaining callers, all of which want a Rails shape the helper does not produce:

- `collection-association.ts:1183`, `:1187` — the `find`-by-id scan; Rails uses
  `r.id.to_s`.
- `collection-association.ts:1427`, `:1433` — the same scan family; the code
  already carries a comment at `:1416-1417` explaining that `primaryKeyValue`
  "returns an _array_ for a composite-PK klass", with a bespoke `normalize`
  helper to paper over it. That comment is the deviation documenting itself.

## Converged shape

Delete `primaryKeyValue` and its `HasManyThroughAssociation` override. At each
call site read `record.id` and apply the shape Rails applies there:

- the `exists?` path keeps Rails' `composite_primary_key?` zip-to-hash (already
  landed in #6743);
- the scan paths use `String(record.id)` the way Rails uses `r.id.to_s`, which
  also retires the bespoke `normalize` helper and its per-element special-casing
  at `:1416-1433`.

Verify the composite arm on a strict engine, not just SQLite: `cpk_books` has a
composite PK whose `id` part is not auto-increment, so CPK regressions in this
area pass on SQLite and red on PostgreSQL/MariaDB.

## Acceptance criteria

- `primaryKeyValue` no longer exists on `CollectionAssociation` or
  `HasManyThroughAssociation`.
- The `normalize` helper at `collection-association.ts:1416` and its
  composite-array special case are gone.
- No new helper replaces either; each call site reads `record.id` inline as
  Rails does.
- `pnpm parity:api:extra --package activerecord` drops the two names; zero new
  rows on `pnpm parity:api:calls` / `:args`.
- Association suites pass on SQLite **and** PostgreSQL/MariaDB.
