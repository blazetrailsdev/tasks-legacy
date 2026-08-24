---
title: "PG visit_AlterTable should append Rails' joined groups instead of sniffing a separator"
status: draft
updated: 2026-08-07
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 70
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced while converging `visit_AlterTable`'s call order in #6206
(`packages/activerecord/src/connection-adapters/postgresql/schema-creation.ts`).

Rails
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/schema_creation.rb:10-15`):

```ruby
def visit_AlterTable(o)
  sql = super
  sql << o.constraint_validations.map { |fk| visit_ValidateConstraint fk }.join(" ")
  sql << o.exclusion_constraint_adds.map { |con| visit_AddExclusionConstraint con }.join(" ")
  sql << o.unique_constraint_adds.map { |con| visit_AddUniqueConstraint con }.join(" ")
end
```

Rails appends each group's parts joined with `" "` and concatenates them onto
whatever `super` produced, with no separator of its own. trails joins within a
group with `", "` and then joins the groups onto `super`'s output with a
separator (`" "` or `", "`) chosen by inspecting the compiled `super` string —
see the `DIVERGENCE (schema_creation.rb:12-14)` comment at the branch and the
`trimmed === \`ALTER TABLE ${table}\`` separator test below it.

The separator sniffing exists because trails' base `visitAlterTable` emits a
comma-separated action list where Rails' emits a differently-shaped string; it
is a symptom of the base visitor's shape, not of this PG override.

## Converged shape

Make the base `visit_AlterTable` (abstract `schema_creation.ts`) produce the
same string Rails' does, then drop the separator sniffing here and append each
group's `" "`-joined string exactly as schema_creation.rb:12-14 does.

## Acceptance criteria

- [ ] The PG `visitAlterTable` body is Rails' four lines: `super`, then three
      `map(...).join(" ")` appends.
- [ ] The `DIVERGENCE` comment and the `trimmed === ...` separator test are gone.
- [ ] PG exclusion-constraint, unique-constraint and validate-constraint
      migration tests stay green with no test renames (PG lane).
