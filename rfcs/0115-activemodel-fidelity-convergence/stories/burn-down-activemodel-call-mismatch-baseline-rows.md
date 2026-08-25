---
title: "Burn down the activemodel call-mismatch baseline rows"
status: in-progress
updated: 2026-08-25
rfc: "0115-activemodel-fidelity-convergence"
cluster: "api-compare"
packages: ["activemodel"]
deps:
  - converge-attribute-methods-copy-on-write-and-alias-helpers
  - converge-attribute-set-builder-residue
  - converge-dirty-tracker-onto-rails-mutation-trackers
deps-rfc: []
est-loc: 300
pr: 7034
claim: "2026-08-25T13:58:43Z"
assignee: "burn-down-activemodel-call-mismatch-baseline-rows"
blocked-by: null
closed-reason: null
---

## Context

`scripts/api-compare/call-mismatches-exclude/activemodel/` holds **54 rows
across 20 shards**:

```text
7  attribute-set.json          7  attribute-methods.json
6  dirty.json                  6  attribute-set/builder.json
4  secure-password.json        3  errors.json
3  callbacks.json              2  validations/numericality.json
2  validations/inclusion.json  2  validations/exclusion.json
2  validations/confirmation.json  2  model.json
1  validations/length.json     1  validations/comparison.json
1  validations/acceptance.json 1  type/helpers/time-value.json
1  type/date.json              1  serializers/json.json
1  attributes.json             1  attribute-registration.json
```

Only 4 carry `kind: "args"` (RFC 0095); the rest are call-set rows (RFC 0047 /
0084). Every row in `model.json` still carries the verbatim RFC 0047 seed
reason:

> Baseline (RFC 0047): wide call-set flag seeded when the wide ratchet landed;
> bucket (b) equivalent or (c) noise pending per-cluster burndown review.

The two `model.json` rows are `initialize` omits `assign_attributes` (converged
by `fan-out-model-serialization-conversion-access-naming-surface`) and
`validates_each` omits `validates_with` (converged by
`fan-out-model-validates-macro-to-validations-validates`).

This story is the **sweep after** the F1–F6 stories: whatever rows the moves
did not strand get converged directly, one shard at a time, sized to the PR LOC
ceiling. It is deliberately last — burning a row that a Phase-1 move is about
to delete is wasted work, and the `attribute-set` / `attribute-methods` /
`dirty` / `builder` shards (26 of the 54 rows) sit in files three Phase-2
stories rewrite.

`arity-exclude.json` has **zero** activemodel rows — nothing to do there.

Also in scope: the `@noRailsEquivalent` ledger. The package carries **192 tags,
181 claiming `PERMANENT` and 11 `CONVERGEABLE`** across `attribute-set.ts`,
`attribute-methods.ts`, `dirty.ts`, `errors.ts`, `serializers/json.ts`,
`attribute-set/builder.ts`, `validations/comparability.ts`,
`validations/numericality.ts`. The 11 CONVERGEABLE claims converge here if no
earlier story took them. A `PERMANENT` claim that describes displaced surface
rather than a TypeScript language shortcoming is re-classified and converged,
not reworded — CLAUDE.md, "A documented deviation is debt, not permission".
There is also 1 `@missingRailsCall` in the package.

## Acceptance criteria

- `call-mismatches-exclude/activemodel` is at **≤ 20 rows**, down from 54.
- Every remaining row carries a reviewed one-line `reason` — no verbatim RFC
  0047 seed strings survive in the activemodel tree.
- Every converged row is **hand-deleted**, sorted position preserved
  (`project_new_baseline_row_must_be_sorted_not_appended`), never via
  `--write` / `parity:api:calls:reseed`; `pnpm parity:api:calls:tighten <shard>`
  run for each stale high-water mark.
- `@noRailsEquivalent` CONVERGEABLE claims in the package reach **0**;
  PERMANENT claims reach ≤ 60.
- `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green;
  `pnpm parity:api:calls:unreviewed` count is at or below its mark.

## Definition of done

Not done if a row is closed by broadening its `reason`, by moving it to
another register, or by adding a sibling row.

## Verification

```bash
API_COMPARE_FORCE=1 pnpm parity:api --calls
pnpm parity:api:calls && pnpm parity:api:calls:args && pnpm parity:api:calls:unreviewed
grep -rc '"key"' scripts/api-compare/call-mismatches-exclude/activemodel/
```
