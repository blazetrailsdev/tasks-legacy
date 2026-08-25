---
title: "converge-arel-build-quoted-model-attribute-unwrapped"
status: draft
updated: 2026-08-25
rfc: "0124-arel-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

`arel-build-quoted-passes-model-attribute-unwrapped` landed as #4882, but
`buildQuoted` still wraps.

`vendor/rails/activerecord/lib/arel/nodes/casted.rb:50-51`'s `when` arm returns
`other` **unwrapped**, so Rails' AST holds the `ActiveModel::Attribute` itself.
`packages/arel/src/nodes/casted.ts:34` still does
`if (other instanceof ModelAttribute) return new BindParam(other)`.

Output SQL agrees — `to_sql.rb:756`'s `add_bind(o)` and `rb:760`'s
`add_bind(o.value)` land the same payload — but the AST shape does not, for any
caller that inspects the node.

`packages/arel/src/nodes/casted.test.ts:98-108` pins today's shape ("wraps an
ActiveModel::Attribute in BindParam") explicitly so the change is visible in the
diff, and its comment cites the landed story as the convergence owner.

Read `project_arel_visit_node_or_value_is_raw_value_path_not_casted` and
`project_arel_build_quoted_passes_model_attribute_unwrapped`'s original PR
before changing the arm — the bind-extraction path is what makes the wrapper
tempting.

## Acceptance criteria

- `buildQuoted` returns an `ActiveModel::Attribute` unwrapped, matching
  `casted.rb:50-51`.
- Prepared-statement bind extraction still lands the same payload
  (`visitBindParam` / `valueForDatabase`) for both shapes.
- `casted.test.ts`'s pinning test is converged to Rails' shape and the stale
  `arel-build-quoted-passes-model-attribute-unwrapped` citation removed.
