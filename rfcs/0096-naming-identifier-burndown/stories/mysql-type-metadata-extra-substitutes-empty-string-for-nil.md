---
title: 'MySQL::TypeMetadata#extra substitutes "" where Rails is nil'
status: claimed
updated: 2026-08-25
rfc: "0096-naming-identifier-burndown"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: "2026-08-25T12:22:53Z"
assignee: "converge-attribute-deep-dup-onto-ruby-dup"
blocked-by: null
closed-reason: null
---

## Context

Rails: `delegate :extra, to: :sql_type_metadata, allow_nil: true`
(`activerecord/lib/active_record/connection_adapters/mysql/column.rb:7`), and
`MySQL::TypeMetadata#initialize` stores `@extra = extra` with a `nil` default
(`.../mysql/type_metadata.rb:13-16`). So Rails' `extra` is `nil` for a column
whose metadata carries none, and `nil` for a Column with no metadata at all.

trails substitutes `""` at both ends:

- `packages/activerecord/src/connection-adapters/mysql/type-metadata.ts` —
  `this.extra = options.extra ?? "";`
- `packages/activerecord/src/connection-adapters/mysql/column.ts` — the
  delegating getter returns `(this.sqlTypeMetadata as TypeMetadata | null)?.extra ?? ""`.

Today nothing observes the difference: every reader compares the value against a
literal (`extra === "auto_increment"`, `mysql/column.rb:18`; the
VIRTUAL/STORED/PERSISTENT regex, `:22-24`; the schema dumper's
generated-column split), and `""` and `nil` both miss. But `""` is truthy-shaped
prose in a port whose Ruby side is `nil`, and it is exactly the
`present?`/`blank?` trap CLAUDE.md names: a future `if extra` arm ported from
Ruby would read the two differently.

Noted while landing `converge-adapter-type-metadata-coder-round-trip`
(PR #7026), which turned `extra` from a Column field into the delegation above.

## Converged shape

`extra` is `string | null`, `null` where Rails is `nil` — on the metadata and
through the Column delegation — and every reader null-guards rather than
leaning on the `""` stand-in.

## Acceptance criteria

- [ ] `MySQL::TypeMetadata#extra` is `null` when no `extra` was given, not `""`.
- [ ] `MySQL::Column#extra` returns `null` with no metadata, mirroring
      `allow_nil: true`.
- [ ] `isAutoIncrement` / `isVirtual` / the MySQL schema dumper still behave
      identically; MySQL/MariaDB lane green.
