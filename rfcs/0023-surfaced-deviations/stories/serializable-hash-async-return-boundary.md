---
title: "Decide serializableHash / asJson return shape across the sync/async boundary"
status: draft
updated: 2026-08-23
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activemodel/src/serialization.ts` carries 67 code lines with no Rails
counterpart — `thenableHash` (`:481`), `asJsonThenable` (`:456`),
`isSerializableCollection` (`:542`), `sendAssociation` (`:524`) — plus the
`SerializableHash = Record<string, unknown> & PromiseLike<...>` type (`:18`) and
the module-private `sync` re-entry parameter on `serializableHash` (`:40`).

The cause is a real async boundary, not a preference. Ruby's
`serializable_add_includes` (`activemodel/lib/active_model/serialization.rb:191`)
reads an association synchronously — `if records = send(association)` — and
`CollectionProxy#to_ary` lazily loads it in-line. In trails an association read
is async, so an `include:`-bearing `serializable_hash` cannot be synchronous.
The Proxy-based thenable is the workaround: sync reads fail loud on an unloaded
include, `await` lazy-loads first.

RFC 0115's story `resolve-serialization-thenable-hash-async-return` is blocked on
this RFC/story existing, because per CLAUDE.md a deviation-convergence story
never closes by writing a better justification, and the alternative — making
`serializableHash` / `asJson` return `Promise` unconditionally — is a
cross-package async-boundary decision spanning activemodel, activerecord,
actionview and activesupport. There is precedent: RFC 0063 made `isValid()`
return `Promise<boolean>` for exactly this reason and the repo treats that as
settled.

### Sync call sites to converge

Non-test producers/consumers of the sync return shape:

- `activemodel/src/serialization.ts` — `serializableHash` (`:38`), its
  `SerializationHost.serializableHash` interface arm (`:142`), the sync
  recursion at `:116` and `:126`, `asJsonThenable` (`:456`).
- `activemodel/src/model.ts:1952` `serializableHash`, `:1958` `asJson`.
- `activemodel/src/serializers/json.ts:90` `asJson`, `:161` `serializableHash`.
- `activerecord/src/serialization.ts:11` `serializableHash` (wraps
  `amSerializableHash` at `:29`).
- `activerecord/src/attribute-methods.ts:888` `serializableHash`.
- `activerecord/src/base.ts:4746` — the `Serialization.serializableHash`
  assignment onto `Base`.
- `activesupport/src/core-ext/object/json.ts` — `asJson`'s dispatcher, which
  calls a value's `asJson`/`serializableHash` and must not assimilate a
  thenable.
- `activerecord/src/relation.ts`, `activerecord/src/token-for.ts`,
  `globalid/src/global-id.ts` — `asJson()` consumers.

Plus every `toJSON` path, which JS's `JSON.stringify` calls synchronously and
which therefore cannot await — that constraint is the crux of the design and
must be answered explicitly by whichever shape this RFC ratifies.

## Acceptance criteria

- A decision is recorded for the return shape of `serializableHash` / `asJson`:
  either (a) `Promise` unconditionally, with every sync call site above
  converged and `toJSON`'s synchronous contract answered, or (b) the thenable
  is ratified with the async-boundary evidence and the `@noRailsEquivalent
PERMANENT` tags cite this story by id.
- If (a): `thenableHash`, `asJsonThenable`, `SerializableHash` and the private
  `sync` re-entry parameter are gone.
- The decision unblocks RFC 0115's `resolve-serialization-thenable-hash-async-return`.
- No third option — the deviation is not closed by broadening a baseline reason.
