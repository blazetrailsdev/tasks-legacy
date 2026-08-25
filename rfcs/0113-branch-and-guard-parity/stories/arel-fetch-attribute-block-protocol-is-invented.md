---
title: "Arel fetchAttribute's boolean-returning block protocol is a trails invention"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: missing-arm
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

Surfaced while porting `Arel.fetch_attribute` in PR #5965.

Ruby's `fetch_attribute(&block)` implementations yield to a block whose
control flow is Ruby's own — `break` aborts the enclosing `each`, and a
`return` in the caller's block (e.g. `where_clause.rb:136-143`
`extract_attribute`) returns from the calling method outright. See
`vendor/rails/activerecord/lib/arel/nodes/nary.rb:21`,
`binary.rb:33`, `grouping.rb:6`, `homogeneous_in.rb:54`, `sql_literal.rb:22`,
`node.rb:155`.

trails' ports instead invent a **boolean-returning callback protocol**: the
block returns `true` to keep traversing and `false` to stop
(`packages/arel/src/nodes/binary.ts:101` `fetchAttributeFromBinary`,
`nodes/nary.ts`, `nodes/grouping.ts`, `nodes/homogeneous-in.ts`). Every caller
must know and honour that convention —
`packages/activerecord/src/relation/where-clause.ts:324` `extractAttribute` and
`packages/activerecord/src/associations/join-dependency/join-association.ts:272`
`nodeReferencesTable` both encode it by hand.

This is invisible in the signature (`(attr: Node) => unknown`), so a caller
that forgets to return `true` silently halts traversal after the first
attribute — a latent bug for any multi-child `Nary` / `HomogeneousIn`
predicate.

## Acceptance criteria

- Audit whether the boolean protocol is load-bearing or whether the node-level
  `fetchAttribute` ports can carry Rails' semantics directly (traverse all
  children; let the caller accumulate), removing the convention.
- If the protocol stays, type it explicitly (`=> boolean`, not `=> unknown`)
  across `Arel.fetchAttribute`, the node implementations, and every caller, and
  document it once at the `Node#fetchAttribute` declaration site.
- No behaviour change: `where-clause`, `merging`, `or`, `and`, `where-chain`,
  and `join-dependency` suites stay green.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
