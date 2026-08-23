---
title: "Resolve serialization's thenableHash dual sync/async return"
status: blocked
updated: 2026-08-23
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps: []
deps-rfc: []
est-loc: 300
priority: null
pr: null
claim: "2026-08-23T14:27:28Z"
assignee: "resolve-serialization-thenable-hash-async-return"
blocked-by: "Blocked on 0023-surfaced-deviations/serializable-hash-async-return-boundary: removing thenableHash/asJsonThenable requires serializableHash and asJson to return Promise unconditionally, a cross-package async-boundary decision (activemodel, activerecord, activesupport, actionview) that also has to answer JSON.stringify's synchronous toJSON contract. Per CLAUDE.md the story cannot close by ratifying the deviation, and the alternative exceeds a bloat-burndown story's scope — it needs its own decision the way RFC 0063 got one for isValid()."
closed-reason: null
---

## Context

`packages/activemodel/src/serialization.ts:473` `thenableHash` (35 code lines)
returns a `Proxy` that behaves as a plain hash **and** as a thenable, so that
`serializableHash` / `asJson` work both synchronously and under `await`. With
`asJsonThenable` (`:449`, 17), `isSerializableCollection` (`:542`) and
`sendAssociation` (`:524`) it is 67 code lines with no Rails counterpart, and a
shape no Rails developer would recognise.

The cause is real: Ruby's `serializable_add_includes`
(`serialization.rb:191`: `if records = send(association)`) reads associations
synchronously; in trails an association read is async, so an `include:`-bearing
`serializable_hash` cannot be synchronous.

## Why this is blocked, not ratified

Per CLAUDE.md, a deviation-convergence story never closes by writing a better
justification. The two real options are:

- **(a)** `serializableHash` / `asJson` return `Promise` unconditionally and
  every sync caller in `activemodel`, `activerecord` and `actionview` is
  converged. There is precedent: RFC 0063 made `isValid()` return
  `Promise<boolean>` for the same reason, and the repo treats that as settled.
- **(b)** the thenable stays and is tagged `@noRailsEquivalent PERMANENT` with
  the async-boundary evidence.

(a) is the recommendation, but it is a cross-package async-boundary decision
that is out of scope for a bloat-burndown story and needs its own RFC — the
same way RFC 0063 got one. This story is registered `blocked` on that RFC
existing.

**Blocker to file before this can move:** an RFC (or a story under
`0023-surfaced-deviations`) covering "serializableHash returns Promise
unconditionally", listing the sync call sites. Grep seed:
`grep -rn "serializableHash\|asJson(" packages/{activemodel,activerecord,actionview}/src --include=*.ts | grep -v test`.

## Acceptance criteria

- `thenableHash`, `asJsonThenable` and their supporting predicates are gone,
  and `serializableHash` / `asJson` have one return shape.
- Or: the async-boundary RFC has explicitly ratified the thenable, and the tag
  cites that RFC by number.
- No third option — do not broaden a baseline reason to close this.
