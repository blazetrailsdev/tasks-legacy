---
title: "AcceptsMultiparameterTime: empty string is truthy in Rails' guard and defaults fill"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
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

Rails' AcceptsMultiparameterTime guard is `return unless values_hash[1] && values_hash[2] && values_hash[3]`
and its defaults fill is `values_hash[k] ||= v`
(`vendor/rails/activemodel/lib/active_model/type/helpers/accepts_multiparameter_time.rb:41-46`).
In Ruby `""` is TRUTHY: an empty string in slots 1-3 passes the guard and then
raises via strict Integer coercion (`Time.utc("", 6, 24)` → ArgumentError:
invalid value for Integer(): ""), and `||=` does NOT replace an empty string
with the default.

Trails' `castFromMultiparameter`
(`packages/activemodel/src/type/helpers/accepts-multiparameter-time.ts`,
post-#5146) treats `""` as absent in both places: the guard's `absent()`
returns null for empty-string date slots, and the defaults loop fills over
`""`. Two trails tests codify this ("empty-string slots get defaults" in
accepts-multiparameter-time-defaults.test.ts). Note the AR extractor converts
blank form values to null before the type sees them
(`attribute_assignment.rb:69` `value.empty? ? nil : ...`), so only direct
type-side casts with `""` diverge.

## Acceptance criteria

- Guard and defaults fill treat `""` as truthy/present per Ruby: empty string
  in slots 1-3 raises ArgumentError (invalid value for Integer()), and
  defaults do not overwrite `""`.
- The trails-only defaults tests are updated to the Rails-faithful behavior.
- AR multiparameter suites (blank handling goes through the extractor's
  blank→nil, so behavior there is unchanged) still pass.
