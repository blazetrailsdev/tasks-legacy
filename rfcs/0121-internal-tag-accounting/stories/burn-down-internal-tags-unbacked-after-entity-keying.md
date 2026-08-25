---
title: "Burn down the ~126 @internal tags the entity-keyed manifest no longer requires"
status: ready
updated: 2026-08-25
rfc: "0121-internal-tag-accounting"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 400
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Follow-up on PR #7057, which made `rails-private-jsdoc` entity-aware so a
private name on one Ruby entity no longer gates a same-named member of a
trails-local helper type sharing the TS file.

That relaxation left ~126 `@internal` tags that the rule no longer requires —
tags PR #7042 and its predecessors added only because the manifest was keyed
file-wide. A sweep of the merged state found them on members of trails-local
helper interfaces and classes, concentrated in:

```text
12  packages/activerecord/src/connection-adapters/abstract/database-statements.ts
 6  packages/arel/src/predications.ts                 (PredicationHost, GroupingFolders, RangePredicates)
 6  packages/activerecord/src/relation/finder-methods.ts
 6  packages/activemodel/src/type/integer.ts
 5  packages/activerecord/src/schema-dumper.ts        (SchemaSource, AdapterSchemaSource)
 5  packages/activerecord/src/connection-adapters/mysql/schema-statements.ts
 5  packages/activemodel/src/type/date.ts
 …  plus i18n/backend/{chain,flatten}.ts, globalid/locator.ts,
    activesupport/{callbacks,messages/serializer-with-fallback,duration/iso8601-parser}.ts
```

They were deliberately NOT removed in #7057: `@internal` drops a member from the
measured surface entirely, so pulling the tag re-enters those names into
`parity:api:extra` — and arel's is a hard gate
(`pnpm parity:api:extra:gate`, RFC 0117), which a bulk removal would turn red.
Each tag needs the same per-site decision RFC 0121's enrollment stories already
make: is the member Rails-backed (keep the tag, now for the right reason), novel
surface to delete, or novel surface that earns a
`@noRailsEquivalent PERMANENT|CONVERGEABLE` receipt?

## Converged shape

One package at a time, in the order the enrollment set grows. For each package:

1. Re-run the sweep against the current manifest — a member is a candidate when
   `files[rel]` lists its name but `entities[rel]` does not list its enclosing
   class/interface.
2. Decide each site: delete the member, keep `@internal` with a receipt, or drop
   the tag.
3. Only then add the package to `unbacked-internal-needs-receipt`'s `files` list
   in BOTH `eslint.config.mjs` and `eslint/rails-private-jsdoc.config.mjs` (the
   set is ONLY-GROW; never remove a package to green a run).

`arel` must be sequenced against `parity:api:extra:tighten`, since its
novel/total marks move with any tag that comes off.

## Acceptance criteria

- [ ] For each package taken on, no `@internal` tag remains whose only backing
      was file-wide manifest keying.
- [ ] `pnpm parity:api:extra:gate` green throughout; arel's marks move DOWN or
      not at all, never up.
- [ ] No `@noRailsEquivalent` reason is written that claims neither PERMANENT
      nor CONVERGEABLE.
- [ ] `Rails API/Test Comparison` and `Lint` green.
