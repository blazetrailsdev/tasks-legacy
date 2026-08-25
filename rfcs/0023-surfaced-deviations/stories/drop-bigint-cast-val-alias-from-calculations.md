---
title: 'Drop the "val" CAST-anchor alias from the calculation projections'
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 270
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`execute_simple_calculation` projects the bare aggregate —
`relation.select_values = [select_value]`
(`vendor/rails/activerecord/lib/active_record/relation/calculations.rb:484`).

trails' `executeSimpleCalculation`
(`packages/activerecord/src/relation/calculations.ts`) adds an `.as("val")`
alias on one path only: when the aggregate is over a bigint column on SQLite,
the compiled SQL is wrapped as
`SELECT CAST("val" AS TEXT) AS "val" FROM (<inner>) AS "_bigint_agg"`
(`wrapBigintAgg`), and `"val"` is the anchor that wrapper reads back. SQLite
returns a lossy JS number for a bigint aggregate because SUM/MIN/MAX over a
computed column has no declared type, so `_maybeEnableSafeIntegers` never fires.

The grouped arms carry the same wrapper (with their own aliases).

## Converged shape

Get the bigint aggregate back as a string without a projection alias Rails does
not have — e.g. drive the cast from the driver/type layer (declare the result
type for the aggregate column so safe-integers engages), or move the CAST into
the projected expression so the outer SELECT wrapper disappears entirely. Either
way `select_values` becomes exactly `[select_value]`.

## Acceptance criteria

- [ ] No `.as("val")` (or grouped equivalent) added purely to anchor a CAST
      wrapper.
- [ ] SQLite bigint sum/min/max still return exact values (the bigint tests in
      `calculations.test.ts` / `calculations.trails.test.ts` stay green).

## Absorbed: `grouped-calc-group-key-alias-invented`

Merged in during the RFC 0023 triage pass (2026-08-18). Original title: "groupedAggregate group_key/val aliases are invented; converge onto ColumnAliasTracker names"

### Context

PR #5074 converged the _aggregate_ alias in
`packages/activerecord/src/relation/calculations.ts:groupedAggregate` onto
Rails' `column_alias_for("#{operation} #{column_name}")` (`sum_credit_limit`,
`count_all`; calculations.rb:537), but two invented aliases remain:

- The **group key** is projected as `AS group_key`
  (calculations.ts:~484 `new Nodes.As(groupNode, new Nodes.SqlLiteral("group_key"))`),
  where Rails aliases each group field via
  `ColumnAliasTracker#alias_for(field.to_s.downcase)` — e.g. `firm_id`,
  `companies_firm_id` — and zips `group_aliases`/`group_fields`
  (vendor/rails/activerecord/lib/active_record/relation/calculations.rb:528-548).
  Rails' tracker also dedupes collisions (`_2` suffix); trails has a ported
  `ColumnAliasTracker` class (calculations.ts:~1260) that groupedAggregate
  never uses.
- `singleAggregate` (calculations.ts:~426) projects `AS "val"`; Rails'
  `execute_simple_calculation` uses no invented alias
  (calculations.rb:465-505).
- `wrapBigintAgg` (calculations.ts:~307) hardcodes both invented names.

Any test asserting grouped-calculation SQL or reading raw result columns sees
non-Rails aliases; `having`-clause select_values merging
(calculations.rb:541 `select_values += self.select_values unless
having_clause.empty?`) is also unported.

### Acceptance criteria

- [ ] groupedAggregate projects group fields with `ColumnAliasTracker`-derived
      aliases (Rails names), not `group_key`; result-row reads updated.
- [ ] Multi-lane green; no parity:test regression.
