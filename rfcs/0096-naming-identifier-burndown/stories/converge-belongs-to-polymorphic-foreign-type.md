---
title: "converge-belongs-to-polymorphic-foreign-type"
status: in-progress
updated: 2026-08-24
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 7014
claim: "2026-08-24T22:45:55Z"
assignee: "converge-activesupport-temporal-receiver-chaining"
blocked-by: null
closed-reason: null
---

## Context

RFC 0096 wave-5 residual finding, split out of
`wave-5-residual-arg-shape-findings` (PR #6929) so the remaining convergence is
owned and scheduled.

`BelongsToPolymorphicAssociation` carries a private `foreignTypeName()` that
duplicates `Reflection#foreignType`
(`packages/activerecord/src/reflection.ts:861-869`), so three call sites pass
`ref:foreignTypeName` where Rails passes `reflection.foreign_type`
(`activerecord/lib/active_record/associations/belongs_to_polymorphic_association.rb`).

The duplicate is not a naming choice. `Association#reflection` is typed — and at
runtime may be — the lightweight `AssociationDefinition`, which has no
`foreignType` member, so the Rails call cannot be spelled until the reflection
type is unified.

## Acceptance criteria

1. `Association#reflection` resolves to a type that carries `foreignType`, at
   the type level and at runtime.
2. `belongs-to-polymorphic-association.ts` calls `reflection.foreignType`
   directly; the private `foreignTypeName()` is deleted.
3. The three `naming` rows for this file are gone from
   `pnpm parity:api:calls:args:report`; no new `call-mismatches-exclude/` row.
4. `pnpm parity:api:calls` and `pnpm parity:api:calls:args` stay green.
