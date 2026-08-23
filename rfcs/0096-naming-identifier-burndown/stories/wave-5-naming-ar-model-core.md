---
title: "Wave 5: burn down the AR-closure naming rows in activerecord model core, migration and validations"
status: ready
updated: 2026-08-23
rfc: "0096-naming-identifier-burndown"
cluster: api-compare
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 90
priority: 11
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

RFC 0096's closing story `naming-gate-flip` is blocked on the AR require-closure
reaching **zero convergeable `naming` rows** (`burndown` +
`module-mixin-receiver`). Waves 1-4 drained the population it was scoped
against; a fresh reading of
`scripts/api-compare/output/call-arg-mismatches.json` (artifact of 2026-08-21,
rendered by `pnpm parity:api:calls:args:report`) shows **107 convergeable rows
still standing inside the closure**, across 67 files. This wave-5 band splits
those 107 into six non-overlapping file sets so the flip has a defined finish
line again.

Rules are unchanged from the RFC's `## Design`:

- **Rename to the Rails identifier, not to a better one.** If Rails writes `o`,
  the TS local is `o`, camelCased per `docs/ruby-ts-conventions.md`.
- **Body-local only.** No behavior change, no public surface change.
- **A row that is really an a1 (argument order) or a3 (invented helper /
  conversion) finding is NOT renamed away.** File it against the RFC that owns
  the file and leave the row. Several rows below look like that shape — e.g.
  `activesupport/notifications.ts#instrument` passes a different argument list,
  not a differently-named one — so read each pair against the vendored Ruby
  before renaming.
- `module-mixin-receiver` rows converge by rewiring to the `this`-typed mixin
  idiom (CLAUDE.md, Module mixins), not by renaming the parameter.

## Rows in this slot

20 rows across 11 files. **File set:** `activerecord/` top-level and `validations/`, `middleware/` — NOT `associations/`, `connection-adapters/`, `relation/`, `encryption/`.

- `activerecord/migration.ts` — 5
  - `migrations`: ruby `ref:version` vs ts `ref:rawVersion`
  - `migrations`: ruby `ref:version,ref:name` vs ts `ref:rawVersion,ref:rawName`
- `activerecord/transactions.ts` — 4
  - `with_transaction_returning_status`: ruby `ref:connection` vs ts `ref:modelClass`
  - `before_commit`: ruby `ref:args` vs ts `ref:options`
- `activerecord/integration.ts` — 3
  - `to_param`: ruby `ref:paramDelimiter` vs ts `ref:delimiter`
  - `cache_version`: ruby `ref:timestamp` vs ts `ref:raw`
- `activerecord/aggregations.ts` — 1 (mmr 1)
  - `composed_of`: ruby `ref:this,ref:partId,ref:reflection` vs ts `ref:modelClass,ref:partId,ref:reflection`
- `activerecord/database-configurations.ts` — 1
  - `build_configs`: ruby `ref:defaultEnv,ref:compact` vs ts `ref:currentEnv,ref:filter`
- `activerecord/enum.ts` — 1
  - `serialize`: ruby `ref:fetch` vs ts `ref:mapped`
- `activerecord/future-result.ts` — 1
  - `execute_or_skip`: ruby `ref:this,ref:instrumenter` vs ts `ref:this,ref:#instrumenter`
- `activerecord/middleware/database-selector/resolver/session.ts` — 1
  - `update_last_write_timestamp`: ruby `ref:now` vs ts `ref:instant`
- `activerecord/validations.ts` — 1 (mmr 1)
  - `raise_validation_error`: ruby `ref:this` vs ts `ref:record`
- `activerecord/validations/absence.ts` — 1
  - `validate_each`: ruby `ref:associationOrValue` vs ts `ref:value`
- `activerecord/validations/presence.ts` — 1
  - `validate_each`: ruby `ref:associationOrValue` vs ts `ref:value`

## Acceptance criteria

1. Every convergeable (`burndown` / `module-mixin-receiver`) `naming` row in the
   file set above is either converged to the Rails identifier, rewired to the
   `this`-typed mixin idiom, or re-filed as an a1/a3 finding against the RFC
   that owns the file — with the re-filed story id named in the PR body.
2. `pnpm parity:api:calls:args:report` shows this slot's convergeable count at
   **zero**; no row in the file set is added to any
   `call-mismatches-exclude/` shard (CLAUDE.md — converge, never ratify).
3. No public surface, method name, field name or behavior changes; the diff is
   locals and parameters only (plus mixin-receiver rewiring where it applies).
4. `pnpm build && pnpm test` green; `pnpm parity:api:calls:args` stays green.

## Notes for the claimer

The per-file counts above are from the 2026-08-21 parity artifact and are
**advisory**. Re-run
`API_COMPARE_FORCE=1 pnpm parity:api --calls && pnpm parity:api:calls:args:report`
at claim time and work from the fresh reading — counts drift as sibling RFCs
touch the same files.
