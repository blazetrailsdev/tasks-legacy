---
title: "Preloader::ThroughAssociation#source_preloaders passes a derived scope, not scope"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `Preloader::ThroughAssociation#source_preloaders` hands the **preloader's
own `scope`** to the nested `Preloader`
(`vendor/rails/activerecord/lib/active_record/associations/preloader/through_association.rb:70-71`):

```ruby
def source_preloaders
  @source_preloaders ||= ActiveRecord::Associations::Preloader.new(
    records: middle_records, associations: source_reflection.name,
    scope: scope, associate_by_default: false).loaders
end
```

trails builds a different scope — `reflectionScope._clone()` with the
`where_clause` emptied (unless `source_type`), merged with `preloadScope`
(`packages/activerecord/src/associations/preloader/through-association.ts:219-276`).
The block comment there justifies it: `throughScope` may already have copied the
whole reflection `where_clause` onto the through query and eager-loaded the
source via `includes!`/`references!` (through_association.rb:117-130), so
re-applying the where here would reference a table this source query never
joins (`no such column: memberships.favorite`).

PR #6130 surfaced this as a wide-baseline row when the method took its Rails
name and became comparable:

```text
scripts/api-compare/call-mismatches-wide-exclude/activerecord/associations/preloader/through-association.json
  source_preloaders -> scope
```

The deviation is **pre-existing and documented**, not introduced by #6130 — but
a baseline row is debt, not permission.

## Why it is not obviously convergeable

Rails passes `scope` because its `through_scope` and `scope` compose
differently than ours: `preloader/association.rb:294-304` builds `scope` from
`reflection_scope` merged with `preload_scope`, and the eager-load branch of
`through_scope` leaves the source query's own predicates intact in a way our
JoinDependency path does not. Converging means understanding why our
eager-loaded middle records need the where emptied where Rails' do not — likely
a `through_scope` fidelity gap, not a `source_preloaders` one.

## Acceptance criteria

- `sourcePreloaders` passes `this.scope` — the same value Rails passes — or the
  investigation lands a `@missingRailsCall` at the call site naming the exact
  Rails-side mechanism that makes the difference, with a `file:line`.
- If the real divergence turns out to be in `throughScope`, fix it there and
  file this one closed against that fix.
- The `source_preloaders -> scope` row is deleted from the wide baseline
  (hand-delete the row; the baseline only shrinks, never `--write`).
- Regression coverage for the case the comment names — a through association
  whose reflection scope predicates the through table (`orderedPostComments` /
  the `memberships.favorite` shape) — verified to FAIL if the where is
  re-applied.

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
