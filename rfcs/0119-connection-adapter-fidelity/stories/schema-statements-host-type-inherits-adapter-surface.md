---
title: "schema-statements-host-type-inherits-adapter-surface"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`packages/activerecord/src/connection-adapters/abstract/schema-statements.ts:313`
declares

```ts
export interface SchemaStatements extends DatabaseAdapter, SchemaQuoter {}
```

Rails has no such type: `schema_statements.rb` is a `module SchemaStatements`
that the adapter `include`s, so the members flow adapter-ward, not the other
way. The TS inheritance runs backwards — it republishes **every required
`AbstractAdapter` member** as public surface of `schema-statements.ts`, whose
Rails counterpart (`connection_adapters/abstract/schema_statements.rb`) does not
declare them.

`pnpm parity:api:extra` measures the result: **159 of activerecord's 1392 extra
names are `connection-adapters/abstract/schema-statements.ts::<adapter member>`**
rows — `commit`, `clearCache`, `beginTransaction`, `buildInsertSql`,
`castBoundValue`, `close`, `delete`, … — none of which the file declares itself.

Optional (`?:`) members are exempt from that count, which makes this an active
ratchet trap rather than inert debt: **converging any abstract-adapter member
from optional to required adds a row and reds
`pnpm parity:api:extra:gate`**, even when the flip is exactly what Rails does.

Measured instance (PR #7041, story
`abstract-adapter-explain-required-not-implemented`): making `explain` required
per `database_statements.rb:180-182` moved activerecord from 1392 → 1393 with
the single new row `schema-statements.ts::explain`. Reverting only that
`?` restored 1392; the `NotImplementedError` base body itself is gate-clean.
That story therefore shipped its base body and had to leave the interface member
optional and the `c.explain!` assertion in `explain.ts:61` in place.

## Acceptance criteria

- [ ] `schema-statements.ts` no longer inherits the adapter's member set — the
      `SchemaStatements` host type expresses what `schema_statements.rb`'s module
      body actually needs from its receiver, the way the other host interfaces in
      `connection-adapters/abstract/` do.
- [ ] The ~159 `schema-statements.ts::<adapter member>` extra rows are gone;
      `pnpm parity:api:extra:tighten` narrows the activerecord marks in the same
      PR (writes DOWN only, no reseed).
- [ ] `explain` becomes a required member on the abstract adapter and
      `explain.ts`'s `execExplain` drops the `!` assertion and its comment,
      closing the deferred half of
      `abstract-adapter-explain-required-not-implemented`.
- [ ] `pnpm parity:api` delta non-negative; SQLite, PostgreSQL and MySQL/MariaDB
      lanes green.
