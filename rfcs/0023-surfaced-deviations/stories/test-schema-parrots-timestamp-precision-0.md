---
title: "TEST_SCHEMA parrots timestamps omit schema.rb's precision: 0; ratchet OPTION_DEBT_CEILING to 0"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

`parity:schema` (PR #5218) now compares column options and reports 4 live
`OPTION` divergences, held under `OPTION_DEBT_CEILING = 4` in
`scripts/schema-compare/compare.ts`:

- `parrots.created_at`, `parrots.created_on`, `parrots.updated_at`,
  `parrots.updated_on`

schema.rb pins `precision: 0` on these timestamps
(`vendor/rails/activerecord/test/schema/schema.rb:893-896`), but TEST_SCHEMA
(`packages/activerecord/src/test-helpers/models/test-schema.ts`, the `parrots`
block) leaves precision implicit, so the introspected columns carry the
adapter-default sub-second precision instead of Rails' bare-second DATETIME.

## Acceptance criteria

- Add `precision: 0` to the four `parrots` timestamp columns in TEST_SCHEMA so
  they mirror schema.rb.
- Verify no timestamp/dirty/optimistic-locking test that uses `parrots`
  regresses on the reduced precision (parrots is heavily exercised).
- Ratchet `OPTION_DEBT_CEILING` down to `0` in
  `scripts/schema-compare/compare.ts` and update its doc comment; the gate then
  flips fatal, so any future option divergence fails CI.
- Update the vendored-schema test in `scripts/schema-compare/compare.test.ts`
  that pins the live count.
