---
title: "Relation#isStrictLoading is Rails' strict_loading_value"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Preloader::Association#cascade_strict_loading` reads the relation's
`strict_loading_value`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/association.rb:310-312`):

```ruby
def cascade_strict_loading(scope)
  preload_scope&.strict_loading_value ? scope.strict_loading : scope
end
```

trails spells that reader `isStrictLoading`, so the ported body reads
`this.preloadScope?.isStrictLoading`
(`packages/activerecord/src/associations/preloader/association.ts:369-371`).
The value is read at the right place with the right semantics — only the name
differs — but the wide call-set ratchet flags the missing
`strict_loading_value` call, and PR #6130 had to baseline it:

```text
scripts/api-compare/call-mismatches-wide-exclude/activerecord/associations/preloader/association.json
  cascade_strict_loading -> strict_loading_value
```

The reader lives on `Relation`, not on the preloader, which is why #6130 did
not converge it in place — the rename is a `Relation`-surface change with its
own call-site sweep.

`strict_loading_value` is a plain Rails attribute reader returning a value, not
a predicate: per `docs/ruby-ts-conventions.md` it should be `strictLoadingValue`,
not `isStrictLoading`. The `is` prefix is reserved for Ruby `?` predicates.

## Converged shape

Rename the `Relation` reader `isStrictLoading` → `strictLoadingValue`, sweep
every call site, and delete the baseline row above (the wide baseline only
shrinks — remove the one row by hand, do not `--write`).

## Acceptance criteria

- `Relation#strictLoadingValue` is the reader's name; `isStrictLoading` is gone
  (not aliased — an alias is new extra surface).
- `cascadeStrictLoading` reads it, and the
  `cascade_strict_loading -> strict_loading_value` row is deleted from
  `call-mismatches-wide-exclude/activerecord/associations/preloader/association.json`.
- `pnpm parity:api:extra --package activerecord` does not gain a row.
- `pnpm typecheck`, `pnpm lint`, and the preloader / strict-loading suites pass.

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
