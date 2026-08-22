---
rfc: "0112-one-rails-thing-n-trails-things"
title: "One Rails thing, N trails things — duplicate bodies and split stores"
status: active
created: 2026-08-18
updated: 2026-08-22
owner: "@deanmarano"
packages:
  - "activerecord"
  - "activemodel"
  - "activesupport"
  - "actionpack"
  - "arel"
  - "date"
  - "globalid"
  - "trailties"
clusters:
  - "duplicate-bodies"
  - "split-stores"
  - "dead-mixin-companions"
  - "wrapper-vs-subclass"
  - "duplicate-definition-gate"
related-rfcs:
  - "0111-error-class-message-parity"
  - "0023-surfaced-deviations"
  - "0006-collection-store-unification"
  - "0026-adapter-layout-fidelity"
  - "0107-relation-ts-decomposition"
priority: 2
---

# RFC 0112 — One Rails thing, N trails things

## Summary

The most repeated pathology in the deviation backlog: Rails keeps **one** body,
or **one** ivar; trails keeps two or three and hand-syncs them. 93 open
`0023-surfaced-deviations` stories describe it — 43 duplicated code, 50
duplicated state — for roughly 14,390 estimated LOC, a fifth of that register.

This is the generalisation of **RFC 0006**, which fixed exactly this shape for
collection stores (`_cachedAssociations` + the proxy's target array → the
association's `@target`), closed after 5 stories, and never came back for the
rest of the repo.

## Motivation

### Duplicated state

| Rails keeps            | trails also keeps                                         |
| ---------------------- | --------------------------------------------------------- |
| `joins_values`         | `_joinClauses`, a raw-SQL/Arel side-channel               |
| `with_values`          | a bespoke `_ctes` array                                   |
| `@statements`          | `_statementPool`, plus a `_statementPoolForTest` accessor |
| `@raw_connection.nil?` | `_permanentlyClosed` **and** `_isFakeConnection`          |
| `Base.configurations`  | `DatabaseTasks.databaseConfiguration`, a second registry  |
| `@association_cache`   | the same map, holding ad-hoc non-`Association` literals   |

### Duplicated code

- `abstract/database-statements.ts` defines each method **twice** — once as a
  free function, once on the mixin.
- `_assign_attributes`: Rails 1, trails 3.
- `SchemaDumper#tables`: Rails one body; trails an async branch plus a
  synchronous fast path that re-walks the tables inline.
- `AssociationScope#_mergeReferencedJoins` hand-reimplements `Merger#merge_joins`
  / `#merge_outer_joins`, which are ported, Rails-faithful, and sitting in
  `relation/merger.ts`.

### Why this is not two RFCs

The two halves cause each other. A duplicated body usually exists **because** a
second store feeds it; a second store survives **because** a second body reads
it. Split into two RFCs, each blocks on the other.

The 2026-08-18 triage pass hit this concretely: retiring the bespoke `_ctes`
array is what puts the already-ported `build_with_value_from_hash` on the live
path at all. Converging one without the other is not smaller, it is impossible.

### Why it is worth a campaign rather than piecemeal fixes

Hand-synced stores drift, and the drift is invisible until it isn't. RFC 0006's
motivation said it first — _"so we stop hand-syncing them"_ — and the same
sentence now applies to at least six more pairs. Every one of them is a place
where a fidelity fix to one copy silently leaves the other behind.

## Design

### Gate: one cheap half, one that needs an extractor change

**Duplicate bodies — cheap.** `parity:api` already matches every TS definition
to its Ruby counterpart; that matching is the whole point of the manifest. A
Ruby method claimed by two or more TS definitions is derivable from the existing
artifact with **no extractor change** — a `duplicate-definitions` ratchet with
the same only-shrink contract as the RFC 0084 / 0095 call gates.

**Split stores — needs work.** `scripts/api-compare/extract-ruby-api.rb` is a
Ripper AST walker that captures methods, params, calls and visibility — **but
not ivars**. Comparing "Rails ivars on this class" against "private fields on
the TS class" means teaching it to collect `@ivar` assignment nodes. The AST is
already in hand so this is tractable, but it is real tooling work and is scoped
as this RFC's **first story**, not assumed.

If the ivar extractor turns out to be noisy — Rails memoisation ivars,
`@__ivar` internals, ivars set only in tests — ship the duplicate-definition
gate alone and run the store half as an ungated campaign. Say so rather than
widening an allowlist until it goes quiet.

### Scope boundary against RFC 0107

Four of the 50 split-store stories live **only** in
`packages/activerecord/src/relation.ts`. That file is RFC 0107's territory —
`active`, priority 1, explicitly about relation.ts decomposition and invented
machinery. **Those four stay with 0107; this RFC takes the other 89.** Without
this line the two campaigns collide on the largest file in the repo.

### Representative stories

| est-loc | story                                                          |
| ------: | -------------------------------------------------------------- |
|     400 | `fold-join-clauses-into-joins-values`                          |
|     400 | `retire-associations-array-for-reflection-registry`            |
|     400 | `current-attributes-port-body`                                 |
|     350 | `consolidate-three-assign-attributes-implementations`          |
|     300 | `with-clause-uses-bespoke-ctes-not-with-values`                |
|     300 | `consolidate-duplicated-through-association-module`            |
|     250 | `database-statements-duplicate-bodies-free-function-and-mixin` |
|     200 | `association-cache-holds-only-association-instances`           |
|     150 | `string-inquirer-wraps-a-value-where-rails-subclasses-string`  |

## Non-goals

- **`relation.ts`-only stores.** RFC 0107 owns them; see the scope boundary.
- **Adapter method _homing_** — a body on `Mysql2Adapter` that Rails puts on
  `AbstractMysqlAdapter`. Six such stories overlap the dead-mixin-companion wave
  and come along for free; the other ~36 belong to closed RFC 0026's
  methodology and should re-open it rather than expand this one.
- **Collapsing sync/async twins as a class.** Where a sync copy exists purely
  because a TS body cannot await, that is RFC 0063 / 0068 / 0087 territory. Only
  twins with no async justification are in scope here.
- **Deleting a store that carries behaviour Rails lacks** without first tracing
  that behaviour to a Rails line. A store is not invented merely because Rails
  has no field by that name.

## Alternatives considered

- **Re-open RFC 0006 and widen it.** Tempting — it is the precedent and the
  title still fits. Rejected because 0006 is `closed` with all 5 stories `done`;
  re-opening a terminal RFC to hold 93 new stories misrepresents what it
  delivered and breaks its own verification claim.
- **Two RFCs, one per half.** Rejected on the mutual-dependency argument above.
- **Fold into RFC 0107.** 0107's framing is one file. 62 of the 93 stories touch
  `activerecord` but only 4 are relation.ts-only; the rest span
  `activesupport`, `activemodel`, `actionpack`, `arel` and three more packages.
- **Gate first, stories later.** The duplicate-definition gate could ship before
  any convergence and would immediately red a large surface. Rejected: a gate
  with no landing path is a blocked CI job, not a plan. Gate lands in Phase 1
  report-only.

## Rollout

Story IDs are assigned when the RFC moves to `active` and the 89 stories
re-home.

1. **Phase 1 — tooling.** Duplicate-definition ratchet over the existing
   `parity:api` artifact, report-only. Ivar collection in
   `extract-ruby-api.rb`, also report-only. Both must land before the noise
   floor can be judged.
2. **Phase 2 — dead mixin companions.** ~10 stories, ~2,060 LOC. A body on the
   concrete class plus a mixed-in companion that never runs. Mechanically
   findable, mostly deletion, and the cheapest evidence that the gate works.
3. **Phase 3 — bespoke parallel stores.** The `_joinClauses` / `_ctes` /
   `_associations` / `_statementPool` family. The highest-value half, and where
   the ivar extractor pays for itself.
4. **Phase 4 — N divergent copies of one method.** ~7 stories, ~1,190 LOC. Each
   is a merge, so each needs the behavioural union of the copies established
   first. Slowest per LOC; deliberately last.
5. **Phase 5 — wrappers where Rails subclasses the real thing.**
   `StringInquirer` wrapping a `_value`, `GlobalID#uri` as a string rather than a
   `URI::GID`, `AdapterSchemaSource` hand-projecting column flags instead of
   passing `Column`s.

## Verification

- The `duplicate-definitions` baseline reaches **0 rows** for the 89 in-scope
  stories' files.
- No Ruby method in the `parity:api` manifest is claimed by more than one TS
  definition outside a reviewed exclusion.
- Named stores are gone repo-wide: `_joinClauses`, `_ctes`, `_statementPool`,
  `_permanentlyClosed`, `_isFakeConnection`, `DatabaseTasks.databaseConfiguration`.
- `pnpm parity:api:extra` novel-name count drops for every file touched — a
  retired duplicate is invented surface leaving the tree.
- All three adapter lanes green throughout; no story converges by adding a
  baseline row.

## Open questions

1. **Is the ivar extractor's signal usable?** Rails uses ivars for memoisation
   (`@composite_query_constraints_list ||= …`) as freely as for state, and a
   memo is not a store. **Recommendation:** collect ivars in Phase 1
   report-only, then decide — and if memo-vs-store cannot be separated
   mechanically, run Phase 3 ungated rather than shipping a gate nobody trusts.
2. **How much of Phase 4 is really a merge?** Three copies of
   `_assign_attributes` may be three copies of the _same_ body, in which case it
   is a deletion, not a reconciliation.
   **Recommendation:** diff the three before sizing; `consolidate-three-assign-attributes-implementations`
   currently carries est-loc 350 on the pessimistic reading, and its sibling
   `activemodel-assign-attribute-still-writes-through-write-attribute` (140)
   should land first — it puts all three on the same shape and materially lowers
   that estimate.
3. **Does the RFC 0107 boundary hold as 0107 progresses?** 0107 is decomposing
   `relation.ts`; a store that is relation.ts-only today may not be next month.
   **Recommendation:** re-check the boundary at each phase transition rather
   than fixing the split once.

## Changelog

- 2026-08-18: initial RFC, carved out of `0023-surfaced-deviations` by the
  backlog triage pass.
