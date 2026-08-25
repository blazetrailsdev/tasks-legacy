---
title: "YAMLColumn scalar dumps use '---\\nstr' not Ruby's '--- str'"
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

PR #5123 converged `YAMLColumn`'s SafeCoder collection dumps with Ruby's
`YAML.dump` by passing `directives: true` to the `yaml` package's stringify
(`packages/activerecord/src/coders/yaml-column.ts` SafeCoder#dump):
`["ok"]` now serializes as `"---\n- ok\n"`, byte-identical to Psych.

Bare scalars still diverge: the `yaml` package emits `"---\nstr\n"`
(marker, newline, scalar) where Ruby's `YAML.dump("str")` emits
`"--- str\n"` (marker, space, scalar). Both parse back identically, but
DB-stored serialized scalar columns are not byte-identical to Rails, and any
future test asserting a scalar dump literal (e.g. persistence_test.rb's
`"--- Have a nice day\n"` updateColumn round-trips) will mismatch.

Rails source: Psych emits scalars inline after the document marker;
`vendor/rails/activerecord/lib/active_record/coders/yaml_column.rb` delegates
to `YAML.dump`. The `yaml` npm package always breaks after `---` when
directives are enabled — converging likely means post-processing the dump for
scalar documents (replace leading `"---\n"` with `"--- "` when the document
root is a scalar) inside SafeCoder#dump.

## Acceptance criteria

- `SafeCoder#dump("str")` returns `"--- str\n"`, matching Ruby `YAML.dump`.
- Collection dumps (`"---\n- ok\n"`) unchanged.
- Round-trip load unchanged; serialized-attribute/store/persistence suites green.
