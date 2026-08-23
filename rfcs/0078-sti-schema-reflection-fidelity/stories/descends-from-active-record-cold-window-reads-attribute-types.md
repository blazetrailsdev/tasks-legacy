---
title: "descends-from-active-record-cold-window-reads-attribute-types"
status: claimed
updated: 2026-08-23
rfc: "0078-sti-schema-reflection-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: "2026-08-23T16:42:28Z"
assignee: "descends-from-active-record-cold-window-reads-attribute-types"
blocked-by: null
closed-reason: null
---

## Context

`descends_from_active_record?` must answer Rails'
`!columns_hash.include?(inheritance_column)` (`inheritance.rb:82-88`) — real
column metadata, so a concrete model carrying only a _virtual/declared_ `type`
attribute is not classified as an STI subclass.

trails#6805 converged the warm half: `isDescendsFromActiveRecord`
(`packages/activerecord/src/inheritance.ts`) reads the reflected columns out of
the schema cache (`cachedColumnsHash`). The cold half is still open, and cannot
be closed from the reader:

- Calling `modelClass.columnsHash()` there triggers `load_schema`, and this
  predicate is reachable **from inside** a schema load — `defineAttributeMethods`
  → `loadSchema` → the relation it builds → `Base._applyStiTypeCondition`
  (`base.ts:2211`) → `isFinderNeedsTypeCondition` → here. `load_schema` memoizes
  only on the way out (`model_schema.rb:534-545`), so the nested call recursed to
  `RangeError: Maximum call stack size exceeded` on all four adapter lanes
  (run 32446311544, `validations/uniqueness-validation` and `core.trails`).
- Rails is no more re-entrant; it simply never re-enters, because its
  `define_attribute_methods` (`attribute_methods.rb:104-125`) builds no relation.
  The trails-only re-entry is the defect, and it lives in the load path.

Two fixes were attempted inside trails#6805 and both failed, which is what
scopes this story to the load path rather than the reader:

1. Calling `columnsHash()` directly — the CI overflow above.
2. Adding a re-entrancy guard to `loadSchema` (`model-schema.ts`) so the nested
   call bails, then calling `columnsHash()` unconditionally. The recursion moved
   rather than stopped: with a per-class guard the overflow reappeared in
   `loadSchemaBangAnchor` → `tableName` → `resolveTableName`, and with a global
   depth counter it reappeared in `underscore`/`resolveTableName` with the
   reader no longer on the cycle at all. A per-class early return also broke
   `columnsHash()` memo identity (`base.trails.test.ts:348`, "an STI subclass's
   own ignoredColumns memoizes per class and reloads with the base"). The
   re-entrancy is therefore not a single edge to guard — the load path calls
   back into itself through table-name resolution as well — and untangling it is
   this story's actual work.

Measured 2026-08-21: a cold `columnsHash()` on a concrete model whose only `type`
is `attribute("type", "string", { virtual: true })` returns `[]` — so once the
re-entry is gone the reader can call `columnsHash()` unconditionally and the
cold window answers correctly with a one-line change.

Until then the reader falls back to `attribute_types` when the cache misses,
which counts a virtual `type` — the same answer `origin/main` gave from
`_attributeDefinitions.has`, so trails#6805 is not a regression, but it is not a
complete convergence either. The omission is declared with `@missingRailsCall`
at the call site.

## Acceptance criteria

- [ ] Schema loading no longer re-enters `descends_from_active_record?` (the
      relation `defineAttributeMethods` builds is the entry point to remove, or
      the load path grows the guard Rails does not need).
- [ ] `isDescendsFromActiveRecord` calls `columnsHash()` unconditionally, and the
      `@missingRailsCall columns_hash` tag on it is deleted.
- [ ] A regression test covers a concrete, **unreflected** model whose only
      `type` is a virtual declared attribute: it must not be treated as an STI
      subclass. The test must fail on the baseline. The reflected half of that
      case already ships in trails#6805 as
      `inheritance.trails.test.ts` → "a virtual type attribute is not an
      inheritance column"; this story adds the cold twin.
