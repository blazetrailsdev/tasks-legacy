---
title: "Wrap raw execute() array binds in OID::Array::Data so PG type_cast needs no bare-Array arm"
status: ready
updated: 2026-08-25
rfc: "0077-quoting-binds-fidelity"
cluster: null
packages: []
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

Surfaced by #6291 (`wrap-binary-and-array-binds-at-source`). Deleting PG's bare
JS-array arm from `type_cast` red the PG lane with `TypeError: can't cast Array`.

Rails' PG `type_cast` has exactly one array arm —
`when OID::Array::Data then encode_array(value)`
(`activerecord/lib/active_record/connection_adapters/postgresql/quoting.rb:177-185`)
— because a pg-ruby bind is always an already-encoded String; the encoder has run
by the time `type_cast` sees it.

node-postgres instead accepts a JS array and encodes it itself, so
`adapter.execute("INSERT INTO pg_arrays (tags) VALUES ($1)", [["one", "two"]])`
(`packages/activerecord/src/adapters/postgresql/array.test.ts:479`, also :493,
:657) reaches `type_cast` unwrapped. #6291 restored
`if (Array.isArray(value)) return typeCastArray.call(this, value);` in
`packages/activerecord/src/connection-adapters/postgresql/quoting.ts` with that
reason at the call site.

Note the _write_ path is already converged — `write-path-bind-array-columns`
(#6293) made `value_for_database` yield the `OID::Array::Data` wrapper. This is
only the raw-`execute(sql, binds)` boundary.

## Converged shape

Wrap array binds in `OID::Array::Data` at the raw-`execute` boundary (or in the
callers above) so `encode_array` runs before `type_cast`, exactly as pg-ruby's
already-encoded String does. Then the bare-`Array` arm has nothing reaching it
and can be deleted, leaving PG's chain arm-for-arm with rb:177-185.

If node-postgres genuinely requires the bare array (rather than the encoded
`{…}` literal `encode_array` produces), `pnpm tasks block` with that evidence.

## Acceptance criteria

- [ ] Array binds reach `type_cast` as `OID::Array::Data`, or the driver
      constraint is documented with a measurement.
- [ ] The bare-`Array` arm in `postgresql/quoting.ts` `typeCast` is deleted.
- [ ] `adapters/postgresql/array.test.ts` passes unchanged on the PG lane.
