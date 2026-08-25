---
title: "Decimal#cast_value is split into a trails-invented _castWithoutScale"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while landing PR #6643 (assertions-activemodel-type-cluster-fourth-pass).

`vendor/rails/activemodel/lib/active_model/type/decimal.rb:60-78` is ONE method:

    def cast_value(value)
      casted_value = \
        case value
        when ::Float   then convert_float_to_big_decimal(value)
        when ::Numeric then BigDecimal(value, precision || BIGDECIMAL_PRECISION)
        when ::String  then begin value.to_d rescue ArgumentError then BigDecimal(0) end
        else
          if value.respond_to?(:to_d) then value.to_d else cast_value(value.to_s) end
        end
      apply_scale(casted_value)
    end

`packages/activemodel/src/type/decimal.ts` splits it into `castValue` plus a
trails-invented private `_castWithoutScale`, which holds the whole case
statement while `castValue` does only the `apply_scale` step. CLAUDE.md's
decomposition rule is explicit: one Rails method is one TS method, and a helper
Rails does not have should not exist. The leading-underscore name is not a Rails
name either, and it is the reason the `else` arm's Rails-recursive
`cast_value(value.to_s)` reads as a call to a different method than Rails makes.

### Converged shape

Inline `_castWithoutScale` back into `castValue` so the body is the case
statement followed by `applyScale(castedValue)`, matching decimal.rb:60-78 arm
for arm. The recursive `else` arm then calls `castValue` — as Rails does — which
also makes the recursion re-apply scale the way Ruby's does.

Note the interaction with `decimal-cast-value-to-s-fallback` (0105): that story
lands the `cast_value(value.to_s)` half of the same `else` arm. Sequence them,
or land them together.

## Acceptance criteria

- `decimal.ts` has no `_castWithoutScale`; `castValue` is the single method
  mirroring decimal.rb:60-78, with Rails' arm order preserved.
- `pnpm parity:api:extra --package activemodel` reports no new surface, and the
  helper's disappearance does not add a `call-mismatches` row.
- Decimal casting behavior is unchanged: `packages/activemodel/src/type/decimal.test.ts`
  and `decimal.trails.test.ts` stay green, as do the AR `type/**` suites.
