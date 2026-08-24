---
title: "reference-definition-columns-named-underscore-columns"
status: draft
updated: 2026-07-28
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

Rails' `ReferenceDefinition` has a private `columns`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/schema_definitions.rb:280-286`),
called by `add` (`:220`), `add_to` (`:234`) and `column_names` (`:292`). trails
names the same method `_columns`
(`packages/activerecord/src/connection-adapters/abstract/schema-definitions.ts:811`,
called at `:717`, `:733`, `:801`).

Surfaced on PR #5480, which ported `ReferenceDefinition#add`. That port converged
7 wide call-mismatch baseline entries, but `add` -> `columns` still flags,
precisely because our `add` calls `_columns()` and the call-set comparison
matches on Rails names. So the remaining baseline entry is not real drift — it is
the underscore.

Check for a collision before renaming: `TableDefinition`/`Table` in the same file
have their own `columns` surface, and `ReferenceDefinition` also has
`columnNames()` / `columnName()`. If the underscore turns out to be load-bearing,
the alternative is to document it and keep the ratchet entry with an accurate
reason string instead of the generic RFC 0047 seed text.

## Acceptance criteria

- [ ] `ReferenceDefinition#_columns` is renamed to `columns` (matching Rails), or
      the reason it cannot be is recorded at the call site.
- [ ] If renamed, the `add` -> `columns` entry is dropped from
      `scripts/api-compare/call-mismatches-wide-exclude/activerecord/connection-adapters/abstract/schema-definitions.json`
      and `pnpm exec tsx scripts/api-compare/lint-call-mismatches-wide.ts` stays green.
- [ ] `pnpm parity:api --package activerecord` shows no new extra surface.

## Triage note (2026-08-18): the baseline path in this body is stale

This story cites `scripts/api-compare/call-mismatches-wide-exclude/…`. **That
tree no longer exists.** RFC 0084 folded the narrow RFC 0044 ratchet and the
wide one into a single gate over a single baseline:
`scripts/api-compare/call-mismatches-exclude/<package>/<tsFile .ts→.json>`,
gated by `pnpm parity:api:calls` (call-set rows) and `pnpm parity:api:calls:args`
(`kind: "args"` rows, RFC 0095).

Look for the row there, under the same `rubyName` / `call` pair. Everything else
in this story — the Rails and trails `file:line` citations, the described
divergence — is unaffected; only the path to the baseline row changed.

Remember the baseline is only-shrink: on converging, delete the one row by hand
(via `serializeBaseline`, sorted — never `--write`/reseed), then
`pnpm parity:api:calls:tighten <package>/<file>.json` for the stale high-water mark.
