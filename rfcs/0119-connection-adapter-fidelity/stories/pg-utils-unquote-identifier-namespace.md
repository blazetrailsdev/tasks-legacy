---
title: "Move PG unquoteIdentifier under the Utils namespace Rails scopes it to"
status: draft
updated: 2026-08-02
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

Rails puts both identifier helpers on the `PostgreSQL::Utils` module:
`Utils.unquote_identifier`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/postgresql/utils.rb`)
and `Utils.extract_schema_qualified_name`, and call sites read
`Utils.unquote_identifier(...)` — e.g. `schema_statements.rb:115`
inside `indexes`.

trails splits them: `extractSchemaQualifiedName` lives inside the
`export namespace Utils` block of
`packages/activerecord/src/connection-adapters/postgresql/utils.ts:47`, but
`unquoteIdentifier` (`utils.ts:63`) and `splitQuotedIdentifier` (`:70`) are bare
top-level exports outside it. Every one of the ~16 non-test call sites therefore
spells the Rails-namespaced call as a bare import.

Surfaced on #5895 (`converge-pg-indexes-body-shape`): porting Rails'
`Utils.unquote_identifier(c.strip.gsub('""', '"'))` verbatim produced
`TS2339: Property 'unquoteIdentifier' does not exist on type 'typeof Utils'`,
and the call site had to be de-namespaced to compile.

## Acceptance criteria

- `unquoteIdentifier` (and `splitQuotedIdentifier`, if Rails scopes it the same
  way — check `utils.rb` first) move inside the `Utils` namespace so call sites
  read `Utils.unquoteIdentifier(...)` as Rails' do.
- All call sites updated; no bare re-export left behind purely for convenience
  unless a cross-package importer needs it, in which case justify at the site.
- `pnpm parity:api` delta non-negative.
