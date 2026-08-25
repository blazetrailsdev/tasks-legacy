---
title: "Drop the invented _requireConstraint guard on PostgreSQL::Table"
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`PostgreSQL::Table` in
`packages/activerecord/src/connection-adapters/postgresql/schema-definitions.ts:556`
guards all six constraint delegators behind a trails-invented
`_requireConstraint(method)` helper, which throws
`` `${method} is not supported by the current schema backend` `` when the
backing schema object lacks the method. The delegators then call through a
non-null assertion (`this._pgSchema.addExclusionConstraint!(...)`), against a
`SchemaStatementsConstraintLike` interface whose members are all optional.

Rails has no such guard. In
`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_definitions.rb:303`,
`Table` delegates straight to `@base`:

```ruby
def exclusion_constraint(...)
  @base.add_exclusion_constraint(name, ...)
end
```

`@base` is always a PG adapter, so the methods always exist and a missing one
would simply raise `NoMethodError`. The optional-member interface plus the
runtime guard is invented surface: it introduces an error class and message
Rails never emits, and the six `!` assertions exist only to satisfy the
optionality the guard implies.

Affected methods: `exclusionConstraint`, `removeExclusionConstraint`,
`uniqueConstraint`, `removeUniqueConstraint`, `validateConstraint`,
`validateCheckConstraint`.

Found while fixing the duplicated `xml` override on the same class (PR #5625,
story red-92de18fb).

## Acceptance criteria

- `_requireConstraint` is deleted from `PostgreSQL::Table`.
- `SchemaStatementsConstraintLike` members become required, or the type is
  replaced by the PG adapter type the delegators actually receive, so the six
  `!` non-null assertions can be dropped.
- The six delegators call `this._pgSchema.<method>(this._pgTableName, ...)`
  directly, mirroring Rails' `@base.<method>(name, ...)`.
- No test asserts on the invented
  "is not supported by the current schema backend" message; if one does, it is
  removed rather than renamed.
- `pnpm parity:api` and `pnpm parity:test` deltas are non-negative.

## Verification

- `pnpm build && pnpm test:types`
- `pnpm vitest run packages/activerecord/src/connection-adapters/postgresql/`
  (PG-gated lanes exercise the constraint delegators)

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
